# gem5_salam_explain

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

## Start Application

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
It defined NonCoherentDMA, Stream DMA 0, and Stream DMA 1.


## Source Code

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

### Step 1 (Set up addresses)
As shown in [define file](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/benchmarks/sys_validation/gemm/gemm_clstr_hw_defines.h), we should specify DMA and matrix addresses.

Then we should load these addresses into our [host code](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/benchmarks/sys_validation/gemm/hw/top.c#L8).

### Step 2 (Copy input)
DMA will perform the copy operation between DRAM and the scratchpad memory. A sample code is provided below.

```c++
*DmaRdAddr  = m1_addr; \\ source address
*DmaWrAddr  = MATRIX1; \\ destination address from previous step
*DmaCopyLen = mat_size;\\ set size for copy
*DmaFlags   = DEV_INIT;\\ Init DMA operation
//Poll DMA for finish
while ((*DmaFlags & DEV_INTR) != DEV_INTR); \\ Wait for interrupt(finishing job)
```

### Step 3 (Start accelerator)

#### Flags
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

#### Interrupt Service Routine (ISR)
In boot code, we setup an Interrupt Service Routine (ISR) in file [isr.c](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/benchmarks/sys_validation/gemm/sw/isr.c). By calling this routine, the accelerator triggers to set up the end of the execution. This will reset the accelerator status to 0x0.

```c++
void isr(void)
{
	printf("Interrupt\n");
	stage += 1;
	*top = 0x00;
	// printf("%d\n", *top);
	printf("Interrupt finished\n");
}
```
 
### Step 4 (Copy output)
We should use the sample code from [step 2](https://github.com/zahrayousefijamarani/gem5_salam_explain/edit/main/README.md#step-2-copy-input), to copy MATRIX3(result of calculation) to m3_addr(destination of the copy).


## Refrences
[1] Rogers S, Slycord J, Baharani M, Tabkhi H. gem5-SALAM: A system architecture for LLVM-based accelerator modeling. In2020 53rd Annual IEEE/ACM International Symposium on Microarchitecture (MICRO) 2020 Oct 17 (pp. 471-482). IEEE.
