## 前言
开发者可以自己开发和不是自己的程序到Solana 区块链。

这有一个例子， [Helloworld example](https://docs.solana.com/developing/on-chain-programs/examples#helloworld) 。通过这个例子可以学到程序如何写、编译、部署和在链上运行。

### Berkley Packet Filter (BPF)
Solana 链上程序通过[LLVM](https://llvm.org/)编译器基础设施编译成可执行和可链接的[ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)格式，其中包含[Berkley Packet Filter (BPF)](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter)的字节码。

因为Solana使用了LLVM编译器基础设施，所以可以用任何能够针对LLVM的BPF后端的编程语言编写程序。Solana目前支持使用 Rust和C/C++ 来写链上程序。

BPF提供了一种高效的指令集，可以在解释过的虚拟机中执行，也可以作为高效的即时编译本机指令执行。

### 内存地址映射
Solana BPF程序所使用的虚拟地址内存映射是固定的，并排列如下
- Program code starts at 0x100000000
- Stack data starts at 0x200000000
- Heap data starts at 0x300000000
- Program input parameters start at 0x400000000

上面的虚拟地址是起始地址，但是程序可以访问内存映射的子集。如果程序试图读或写一个它没有被授予访问权限的虚拟地址，程序将会panic，并返回一个 ```AccessViolation``` 错误，该错误包含了地址和试图违例的大小。

### 堆栈
BPF uses stack frames instead of a variable stack pointer. Each stack frame is 4KB in size.

If a program violates that stack frame size, the compiler will report the overrun as a warning.

For example: Error: Function _ZN16curve25519_dalek7edwards21EdwardsBasepointTable6create17h178b3d2411f7f082E Stack offset of -30728 exceeded max offset of -4096 by 26632 bytes, please minimize large stack variables

The message identifies which symbol is exceeding its stack frame but the name might be mangled if it is a Rust or C++ symbol. To demangle a Rust symbol use rustfilt. The above warning came from a Rust program, so the demangled symbol name is:

```
$ rustfilt _ZN16curve25519_dalek7edwards21EdwardsBasepointTable6create17h178b3d2411f7f082E
curve25519_dalek::edwards::EdwardsBasepointTable::create
```
To demangle a C++ symbol use c++filt from binutils.

The reason a warning is reported rather than an error is because some dependent crates may include functionality that violates the stack frame restrictions even if the program doesn't use that functionality. If the program violates the stack size at runtime, an AccessViolation error will be reported.

BPF stack frames occupy a virtual address range starting at 0x200000000.


