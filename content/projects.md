## Not so kool projects


### PDM extension header
Implemented Performance and Diagnostic Metrics(PDM) Extension header for IPv6 - [RFC8250](https://datatracker.ietf.org/doc/rfc8250/), for linux using eBPF and Traffic Control(tc). Currently working with the authors on implementing the [PDMv2](https://www.ietf.org/archive/id/draft-ietf-ippm-encrypted-pdmv2-05.txt) Internet Draft. As it involves encryption, and eBPF has limits on no of instructions it's a bit tricky(If you're doing something similar, happy to chat). 

### Energy estimator for kubernetes pods

A set of different programs that work together to run software based power/energy estimation models to measure the energy consumed by different workloads, hardware and software components in a machine.

- librapl: A C/C++ library that provides APIs to access the Running Average Power Limit(RAPL) interfaces in Intel processors on a linux OS. 
- exporter: A prometheus exporter, which exports certain metrics like hardware Performance Counters(HPCs), data from processes, resource usage information, etc.
- estimator: Runs a energy estimation model using the raw metrics exported by the exporter. Implemented [HaPPY](https://www.usenix.org/system/files/conference/atc14/atc14-paper-zhai.pdf) power model and extended it.


### PTRACE-debugger

A simple command line debugger similar to GDB which uses ptrace syscall in linux, which can parse, launch, halt and execute elf binaries. developed to understand how setting breakpoints on memory addresses and source code lines work.