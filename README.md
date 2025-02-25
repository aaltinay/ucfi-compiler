# ucfi-compiler

To use uCFI you also need [ucfi-kernel](https://github.com/uCFI-GATech/ucfi-kernel) and [ucfi-monitor](https://github.com/uCFI-GATech/ucfi-monitor).

## Introduction

ucfi-compiler compiles project source code into a hardened version, so that ucfi-monitor can protect its execution from control-flow hijacking attacks. It contains three components: LLVM pass, X86 backend and ptwrite emulator.

#### LLVM pass

The related code is in `llvm/lib/Transforms/NewCPSensitivePass`. The LLVM pass will complete the follows tasks:

1. Identify and instrument constraining data so that the monitor knows its value

2. Instrument each sensitive basic block (containing at least one control-instruction) to dump its unique ID

3. Change function attribute to avoid tail call optimization

4. Replace all indirect function call with a direct call to a specific function, where an indirect jump helps reach the real target

#### X86 backend

The related code is in `llvm/lib/Target/X86/X86AsmPrinter.cpp`. The X86 backend achieves two tasks

1. Redirect each RET instruction to an dedicated RET instruction

2. Provide a simple parallel shadow stack implementation proposed in [1].

   Note: ucfi just adopts parallel shadow stack to demonstrate the compatibility with shadow stack solutions. We do not claim any contribution or guarantee on protecting the return address. All design novelties go to its orignal authors. This implementation is just a simple version written by ucfi authors. Any implementation bugs go to ucfi authors. Due to implementation difference, overhead of parallel shadow stack could be different from that reported in the original paper.

#### ptwrite emulator

The related code is in `ptwrite-emulator`. ptwrite emulator helps dump arbitrary value (even non-control-flow data) into Intel PT trace. It mainly supports two features

1. A dedicated RET instrution to help achieve return for all functions

2. A dedicated JMP instruction to help achieve indirect function call

3. A code region of all RET instructions to achieve the ptwrite emulation

4. Help setup the parallel shadow stack

## Build Instructions

1. git pull this repo to local system

    `git pull git@github.com:uCFI-GATech/ucfi-compiler.git`
    
    `cd ucfi-compiler`
    
2. create build and install folder

    `mkdir {build,install}`
    
    `cd build`
    
3. configuration the compilation

    `cmake -DCMAKE_INSTALL_PREFIX=../install -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD=X86 ../llvm/`
    
4. build the compiler and install it

    `make -j 8`
    
    `make install`
    
     Now you should have llvm & clang toolchains available in the `install` folder. run `source shrc` in the root directory to add the installation folder into `PATH`

5. build the ptwrite emulator

   `#suppose you are under the root directory of this project`
   
   `cd ptwrite-emulator`
   
   `# if you want to use parallel shadow stack`
   
   `./GenPTWriteFile.py ss`
   
   You will get `pt_write_sim_ss.o` here, which is necessary to be linked into the final executable. Check [Compile a hello world to see how to use it](README.md#compile-a-hello-world).
   
   `# if you do not want to use parallel shadow stack`
   
   `./GenPTWriteFile.py`
   
   You will get `pt_write_sim.o` here, which is necessary to be linked into the final executable.

## Compile a hello world

Currently, ucfi-compiler requires to work on one LLVM IR file of the whole project. In our work, we use [wllvm](https://github.com/travitch/whole-program-llvm) to generate the whole-program-LLVM-IR first, and then we use ucfi-compiler to generate the hardend executable and other auxiliary files.

You can find how to use `wllvm` to generate the whole-program-LLVM-IR following [this link](https://github.com/travitch/whole-program-llvm). Of course, you can use other ways to generate this file, like through [llvm-link](http://llvm.org/docs/CommandGuide/llvm-link.html), or [linking-time-optimization](https://llvm.org/docs/LinkTimeOptimization.html). 

Suppose you have successfully get the one LLVM IR file, here are the instructions to generate the hardened binary. 

0. Lower any indirect jump to if-else branch. Currently uCFI only handles indirect function calls. For indirect jump, we rely on the lowerswitch pass of opt to change them to if-else + direct jump. 

    `opt -lowerswitch the-whole-project-ir-file -o the-whole-project-ir-file-A`
    
    `cp the-whole-project-ir-file-A the-whole-project-ir-file`

1. If you do not want to use shadow stack

    `clang++ -Xclang -load -Xclang ~/pt-cfi/install/lib/LLVMCPSensitivePass.so -Xclang -add-plugin -Xclang -CPSensitive -mllvm -redirectRet /path/to/pt_write_sim.o the-whole-project-ir-file -o hardened-bin`
    
2. If you want to use shadow stack

    `clang++ -Xclang -load -Xclang ~/pt-cfi/install/lib/LLVMCPSensitivePass.so -Xclang -add-plugin -Xclang -CPSensitive -mllvm -redirectRet /path/to/pt_write_sim_ss.o -mllvm -shadowstack the-whole-project-ir-file -o hardened-bin`
    
At this stage, you should get the hardened binary, and the IR file named like `*_pt.bc`. You need both to run the monitor. See the `ucfi kernel` repo for an example running script.
    
## Paper

Hong Hu, Chenxiong Qian, Carter Yagemann, Simon Pak Ho Chung, William R. Harris, Taesoo Kim, and Wenke Lee. 2018. Enforcing Unique Code Target Property for Control-Flow Integrity. In Proceedings of the 2018 ACM SIGSAC Conference on Computer and Communications Security (CCS '18). ACM, New York, NY, USA, 1470-1486. DOI: https://doi.org/10.1145/3243734.3243797

```
@inproceedings{hu:ucfi,
  title        = {{Enforcing Unique Code Target Property for Control-Flow Integrity}},
  author       = {Hong Hu and Chenxiong Qian and Carter Yagemann and Simon Pak Ho Chung and William R. Harris and Taesoo Kim and Wenke Lee},
  booktitle    = {Proceedings of the 25th ACM Conference on Computer and Communications Security (CCS)},
  month        = oct,
  year         = 2018,
  address      = {Toronto, ON, Canada},
}
```

## Authors

[Hong Hu](https://www.cc.gatech.edu/~hhu86/)<br />
[Chenxiong Qian](https://0-14n.github.io/)<br />
[Carter Yaggemann](https://carteryagemann.com/)<br />
Simon Pak Ho Chung<br />
[William R. Harris](https://galois.com/team/bill-harris/)<br />
[Taesoo Kim](https://taesoo.kim/)<br />
[Wenke Lee](http://wenke.gtisc.gatech.edu/)

## Contacts (Gmail)

Hong Hu: huhong789<br />
Chenxiong Qian: chenxiongqian<br />
Carter Yagemann: carter.yagemann

## References

[1] Thurston H.Y. Dang, Petros Maniatis, and David Wagner. 2015. The Performance Cost of Shadow Stacks and Stack Canaries. In Proceedings of the 10th ACM Symposium on Information, Computer and Communications Security (ASIA CCS '15). ACM, New York, NY, USA, 555-566. DOI: https://doi.org/10.1145/2714576.2714635
