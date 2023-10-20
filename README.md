# gem5_salam_explain

## Full system architecture
![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/c618b3e5-8139-41fa-8287-b48823473370)

## Code explain

Code is available in [hwacc](https://github.com/TeCSAR-UNCC/gem5-SALAM/blob/main/src/hwacc/) folder. 

### communication interface

A Communications Interface, shown in Figure, provides access to the system interfaces of gem5 for the purposes of **memory access**, **control**, and **synchronization**. It accomplishes this by providing three basic interfaces in its API: Memory-Mapped Registers (**MMRs**), **memory master ports**, and **interrupt lines** [1]. CommInterface serves as the general system interface for hardware accelerators. It provides a set of memory-mapped registers, as well as master ports for accessing both local busses/SPMs and system memory.

![image](https://github.com/zahrayousefijamarani/gem5_salam_explain/assets/45602698/807500a4-f426-4abb-b856-2f52ea28a442)

It implemented send and receive(TimingResp, ReqRetry) functions also read and write to the ports for
- CommInterface::MemSidePort
- CommInterface::SPMPort
- CommInterface::RegPort
- CommInterface (not specified)

It puts read and write requests in a queue so it has enqueueRead and enqueueWrite.





## Refrences
[1] Rogers S, Slycord J, Baharani M, Tabkhi H. gem5-SALAM: A system architecture for LLVM-based accelerator modeling. In2020 53rd Annual IEEE/ACM International Symposium on Microarchitecture (MICRO) 2020 Oct 17 (pp. 471-482). IEEE.
