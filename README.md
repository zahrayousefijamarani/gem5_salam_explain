# gem5 SALAM 

## Build

### Dependencies
```
sudo apt install build-essential git m4 scons zlib1g zlib1g-dev \
    libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev \
    python3-dev python-is-python3 libboost-all-dev pkg-config

sudo apt install llvm-9 llvm-9-tools clang-9
```

### Clone
```
git clone https://github.com/TeCSAR-UNCC/gem5-SALAM
```

### Build
```
scons build/ARM/gem5.opt -j9
```

## Full system architecture
![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/c618b3e5-8139-41fa-8287-b48823473370)

## Code explain

Code is available in [hwacc](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/src/hwacc/) folder. 

### Communication interface

A Communications Interface, shown in Figure, provides access to the system interfaces of gem5 for the purposes of **memory access**, **control**, and **synchronization**. It accomplishes this by providing three basic interfaces in its API: Memory-Mapped Registers (**MMRs**), **memory master ports**, and **interrupt lines** [1]. CommInterface serves as the general system interface for hardware accelerators. It provides a set of memory-mapped registers, as well as master ports for accessing both local busses/SPMs and system memory.

![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/807500a4-f426-4abb-b856-2f52ea28a442)

It implemented send and receive(TimingResp, ReqRetry) functions and also read and write to the ports for
- CommInterface::MemSidePort
- CommInterface::SPMPort (for scratchpad memory)
- CommInterface::RegPort
- CommInterface (not specified)

It puts read and write requests in a queue so it has enqueueRead and enqueueWrite.

### Accelerator Cluster
A cluster of accelerators should be defined that stores and defines helper functions for the following:
1. System Cache Parameter
2. Local bus
3. SPM and connections
4. DMA connections
Python code for [AccCluster](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/src/hwacc/AccCluster.py).


### Stream DMA
- tick function that checks is there any pending write or read, and then perform them in FIFO order.
- read and write functions that work with memreg.
- streamRead and streamWrite that work with FIFO.
- some other helper functions like status function.
______

## Start Application
Some of the images and explanations are from [2].
We will use the DMA model.
![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/e2279940-91c9-4d27-bd7f-7a3003e3203f)


## Python code for starting
We can find the starting code in [HWAcc](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/configs/SALAM/HWAcc.py) file.
We have the following steps:
1. Create a new Accelerator Cluster ([Explained](https://github.com/zahrayousefijamarani/gem5_salam_explain/edit/main/README.md#accelerator-cluster))
2. Add accelerator to the cluster
3. Add DMA devices to the cluster and connect them

### Step 1 (Create cluster)
After defining AccCluster we should specify the [Accelerator range](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/configs/SALAM/HWAcc.py#L20C9-L20C9). Any accesses to this range from the host code or accelerator are routed to the accelerator cluster.

### Step 2 (Add accelerator)
for adding an accelerator, we should define a [commIntefrace](https://github.com/zahrayousefijamarani/gem5_salam_explain/edit/main/README.md#communication-interface) as below:
```c++
system.acctest.acc = CommInterface(devicename=options.accbench)
```
Then we should do the followings:
1. Setup config for the accelerator([link](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/configs/SALAM/HWAcc.py#L31))
2. Add an SPM for the accelerator([link](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/configs/SALAM/HWAcc.py#L36))
3. Connect the accelerator to the system's interrupt controller([link](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/configs/SALAM/HWAcc.py#L40C11-L40C11))
4. Connect HWAcc to cluster buses([link](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/configs/SALAM/HWAcc.py#L43))

### Step 3 (Add DMAs)
It defined NonCoherentDMA, Stream DMA 0, and Stream DMA 1. Not all kinds of DMAs used for an application.

Stream DAM includes a control pio. This can be written to by top to control from where in the DRAM data is being streamed. The out port of the StreamDMA engine is wired up to the stream ports of one of the accelerators. Each stream is a single input-single output **FIFO**. Each accelerator has a .stream interface into which all the required streams are wired in. In this case i) we read from the DRAM and send data to accelerator S1. ii) read data from accelerator S3 and write it into stream DMA.

## Source Code for running an application

We should write two parts:
1. The code that we want to accelerate (algorithm with any compiler optimization).
2. Host code of accelerator (computation model of accelerator).

We can start with the Generic Matrix Multiply operation (GEMM) application. It takes 2 matrices as input, multiplies them, and stores the result in another matrix.

### Host code
For the [host code](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/benchmarks/sys_validation/gemm/hw/top.c), we should do the following:

1. Set up addresses for scratchpad.
2. Copy data from DRAM into scratchpad. (INPUT)
3. Start accelerator.
4. Copy data from scratchpad to DRAM. (OUTPUT)

#### Step 1 (Set up addresses)
As shown in [define file](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/benchmarks/sys_validation/gemm/gemm_clstr_hw_defines.h), we should specify DMA and matrix addresses.

Then we should load these addresses into our [host code](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/benchmarks/sys_validation/gemm/hw/top.c#L8).

#### Step 2 (Copy input)
DMA will perform the copy operation between DRAM and the scratchpad memory. A sample code is provided below.

```c++
*DmaRdAddr  = m1_addr; \\ source address
*DmaWrAddr  = MATRIX1; \\ destination address from previous step
*DmaCopyLen = mat_size;\\ set size for copy
*DmaFlags   = DEV_INIT;\\ Init DMA operation
//Poll DMA for finish
while ((*DmaFlags & DEV_INTR) != DEV_INTR); \\ Wait for interrupt(finishing job)
```

#### Step 3 (Start accelerator)

##### Flags
To start the accelerator, we set its flag to DEV_INIT:
```
*GEMMFlags = DEV_INIT;
```
There are 4 possible values for the accelerator:
- 0x0: inactive
- 0x1: to start the accelerator.
- 0x4 active.

Then we should [wait](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/benchmarks/sys_validation/gemm/hw/top.c#L33) for the accelerator to acknowledge starting as shown in the image below:

![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/54d2e724-b707-4d05-bde7-be1980a317a6)

##### Interrupt Service Routine (ISR)
In boot code, we setup an Interrupt Service Routine (ISR) in file [isr.c](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/benchmarks/sys_validation/gemm/sw/isr.c). By calling this routine, the accelerator triggers to set up the end of the execution. This will reset the accelerator status to 0x0.

```c++
void isr(void)
{
	printf("Interrupt\n");
	stage += 1;
	*top = 0x00;
	printf("Interrupt finished\n");
}
```
 
#### Step 4 (Copy output)
We should use the sample code from [step 2](https://github.com/zahrayousefijamarani/gem5_salam_explain/edit/main/README.md#step-2-copy-input), to copy MATRIX3(the result of calculation) to m3_addr(destination of the copy).

### Algorithm 
The code for gemm application can be found in [gemm.c](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/benchmarks/sys_validation/gemm/hw/gemm.c) 

In this code we have loop for multiply, and **#pragma**.
#### Pragma
To expose parallelism for computation and memory access we fully unroll a loop of the application. The simulator will natively pipeline the other loop instances for us. To accomplish the loop unrolling we can utilize clang compiler pragmas such as 
```
#pragma clang loop unroll_count(8) \\ 8 way parallelism
```
This will do operations in parallel as much as it can.

#### Rules
1. **SINGLE FUNCTION** Only a single function is permitted per accelerator .c file.
2. **NO LIBRARIES** Cannot use standard library functions. Cannot call into other functions
3. **No I/O** No prints or writes to files. Either use traces or write back to cpu memory to debug
4. **Only locals** or args Can only work with variables declared within a function or input arrays.

### Configs
For each of our accelerators, we also need to generate an yaml file. In each yaml file, we can define the number of cycles for each IR instruction and provide any limitations on the number of Functional Units (FUs) associated with IR instructions. These files can be found in [configs](https://github.com/TeCSAR-UNCC/gem5-SALAM/tree/main/benchmarks/sys_validation/gemm/configs) folder.

## Other type of applications
- Batch: We can break matrices into two parts(batch) and then write [code](https://github.com/zahrayousefijamarani/gem5_salam_explain/edit/main/README.md#host-code) twice.
![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/55f18a05-be81-449e-bb98-4e742a72e2ba)
- Cache: The cache model hooks up the accelerator to the global memory through a coherence crossbar. With coherence available the accelerators can directly reference the DRAM space mapped to the CPU. 
![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/539fb31b-0d12-483a-aa49-fba58570b34a)
- Multi-accelerator: We can define multiple accelerators and each of them performs a specific operation.
![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/f4d98f13-aba3-4c13-8e28-4ceaf6562a20)

## Stream
### stream arch
![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/dfe0ef83-619a-4aa0-8c5e-9649546a4324)

### stream DMA
![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/cdc12a75-d3f6-40d8-aa55-e7f6e7eb05b3)

In the host code(top.c), we should connect DRAM to FIFO port.
```c++
*StrDmaRdAddr = in_addr;
*StrDmaRdFrameSize = INPUT_SIZE; // Specifies number of bytes
*StrDmaNumRdFrames = 1;
*StrDmaRdFrameBuffSize = 1;
// Start Stream
*StrDmaFlags = STR_DMA_INIT_RD | STR_DMA_INIT_WR;
```

### stream buffer
Stream buffers establish ports directly between accelerators. 

## Refrences
[1] Rogers S, Slycord J, Baharani M, Tabkhi H. gem5-SALAM: A system architecture for LLVM-based accelerator modeling. In2020 53rd Annual IEEE/ACM International Symposium on Microarchitecture (MICRO) 2020 Oct 17 (pp. 471-482). IEEE.

[2] https://www.cs.sfu.ca/~ashriram/Courses/CS7ARCH/tutorials/gem5-acc/index.html
