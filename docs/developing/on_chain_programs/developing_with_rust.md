## Rust 开发
### 项目布局
Solana Rust项目遵循典型的[Rust项目布局](https://doc.rust-lang.org/cargo/guide/project-layout.html):
```
/inc/
/src/
/Cargo.toml
```
但是Rust还需要include
```
/Xargo.toml
```
并且需包含
```
[target.bpfel-unknown-unknown.dependencies.std]
features = []
```
在进行跨程序调用时，为了获得对指令助手的访问，Solana Rust程序可能直接相互依赖。在这样做时，重要的是不要拉入依赖程序的入口点符号，因为它们可能与程序自己的冲突。为了避免这种情况，程序应该在Cargo.toml中定义一个exclude_entrypoint特性和使用这个 exclude entrypoint。

[Define the feature](https://github.com/solana-labs/solana-program-library/blob/a5babd6cbea0d3f29d8c57d2ecbbd2a2bd59c8a9/token/program/Cargo.toml#L12)
[Exclude the entrypoint](https://github.com/solana-labs/solana-program-library/blob/a5babd6cbea0d3f29d8c57d2ecbbd2a2bd59c8a9/token/program/src/lib.rs#L12)
然后，当其他程序将此程序作为依赖项包括进来时，它们应该使用exclude_entrypoint特
[Include without entrypoint](https://github.com/solana-labs/solana-program-library/blob/a5babd6cbea0d3f29d8c57d2ecbbd2a2bd59c8a9/token-swap/program/Cargo.toml#L19)

### 项目依赖项
一个最小化的Solana Rust 程序必须 pull [solana-program](https://crates.io/crates/solana-program) crate。
Solana BPF程序有一些限制，可能会阻止包含一些crate作为依赖或需要特殊处理。
例如：

- 需要体系结构的crate是官方工具链所支持的crate的子集。对此没有任何变通方法，除非将crate分叉，并将BPF添加到那些架构检查中。
- crate可能依赖于rand，这在Solana的确定性程序环境中是不支持的。包含依赖crate指的是依赖Rand。
- crate可能会溢出堆栈，即使堆栈溢出代码没有包含在程序本身。更多信息请参考[堆栈](#堆栈)。

### 如何构建
先安装环境
- 安装最新稳定版本的Rust [https://rustup.rs/](https://rustup.rs/)
- 安装最新版本的Solana命令行工具 [https://docs.solana.com/cli/install-solana-cli-tools](https://docs.solana.com/cli/install-solana-cli-tools)

普通的cargo构建可用于构建针对主机的程序，可以用于单元测试:
可以在主机上使用普通的cargo构建方式构建程序，用于单元测试：
```
$ cargo build
```

编译一个可部署到集群中的Solana BPF目标构建特定程序，如SPL Token:
```
$ cd <the program directory>
$ cargo build-bpf
```

### 如何测试
Solana 程序可以使用传统的```cargo test```命令来进行单元测试，直接执行程序函数。

为了方便在更接近活动集群的环境中进行测试，开发人员可以使用 [program-test](https://crates.io/crates/solana-program-test) crate. ```program-test``` carte 启动运行时的本地实例，并允许测试发送多个事务，同时在测试期间保持状态。

这里面 [test in sysvar example](https://github.com/solana-labs/solana-program-library/blob/master/examples/rust/sysvar/tests/functional.rs) 展示了一个包含syavar帐户的指令是如何被程序发送和处理的。

### 程序入口(Program Entrypoint)
程序导出一个已知的入口点符号，Solana运行时在调用程序时查找并调用该符号。Solana支持多个版本的BPF加载器，它们之间的入口点可能有所不同。程序必须编写并部署到同一个加载程序中。
更多信息请参见[前言](##开发###链上程序####前言)。

当前支持两个加载器 [BPF Loader](https://github.com/solana-labs/solana/blob/7ddf10e602d2ed87a9e3737aa8c32f1db9f909d8/sdk/program/src/bpf_loader.rs#L17) 和 [BPF loader deprecated](https://github.com/solana-labs/solana/blob/7ddf10e602d2ed87a9e3737aa8c32f1db9f909d8/sdk/program/src/bpf_loader_deprecated.rs#L14)

它们都有相同的原始入口点定义，下面是运行时查找和调用的原始符号:
```
#[no_mangle]
pub unsafe extern "C" fn entrypoint(input: *mut u8) -> u64;
```
这个入口点接受一个通用的字节数组，其中包含序列化的程序参数(程序id、帐户、指令数据等)。为了反序列化参数，每个加载器都包含自己的包装器宏，该宏导出原始入口点，反序列化参数，调用用户定义的指令处理函数，并返回结果。

可以在下面的地址找到入口宏：
- [BPF Loader's entrypoint macro](https://github.com/solana-labs/solana/blob/7ddf10e602d2ed87a9e3737aa8c32f1db9f909d8/sdk/program/src/entrypoint.rs#L46)
- [BPF Loader deprecated's entrypoint macro](https://github.com/solana-labs/solana/blob/7ddf10e602d2ed87a9e3737aa8c32f1db9f909d8/sdk/program/src/entrypoint_deprecated.rs#L37)

程序定义的入口点宏调用的指令处理函数必须是这种形式:
```
pub type ProcessInstruction =
    fn(program_id: &Pubkey, accounts: &[AccountInfo], instruction_data: &[u8]) -> ProgramResult;
```

参考[helloworld's use of the entrypoint](https://github.com/solana-labs/example-helloworld/blob/c1a7247d87cd045f574ed49aec5d160aefc45cf2/src/program-rust/src/lib.rs#L15) 例子看是如何一起使用的

### 参数反序列化
每个加载器都提供一个帮助函数，将程序的输入参数反序列化为Rust类型。入口点宏自动调用反序列化助手:
- [BPF Loader deserialization](https://github.com/solana-labs/solana/blob/7ddf10e602d2ed87a9e3737aa8c32f1db9f909d8/sdk/program/src/entrypoint.rs#L104)
- [BPF Loader deprecated deserialization](https://github.com/solana-labs/solana/blob/7ddf10e602d2ed87a9e3737aa8c32f1db9f909d8/sdk/program/src/entrypoint_deprecated.rs#L56)

有些程序可能希望自己执行反序列化，它们可以通过提供自己的[raw entrypoint](https://docs.solana.com/developing/on-chain-programs/developing-rust#program-entrypoint)实现来实现。请注意，所提供的反序列化函数保留了对允许程序修改的变量(lamports、帐户数据)的序列化字节数组的引用。这样做的原因是，当返回时，加载器将读取这些修改，因此它们可能被提交。如果程序实现了自己的反序列化函数，则需要确保程序希望提交的任何修改都被写回输入字节数组中。

关于加载器如何序列化程序输入的详细信息可以在 [Input Parameter Serialization](https://docs.solana.com/developing/on-chain-programs/overview#input-parameter-serialization)文档中找到。

### 数据类型
加载器的入口点宏使用以下参数调用程序定义的指令处理器函数:
```
program_id: &Pubkey,
accounts: &[AccountInfo],
instruction_data: &[u8]
```
program_id 是当前执行程序帐户的pubkey。
帐户是指令引用的帐户的有序片段，并表示为AccountInfo结构。帐户在数组中的位置表示其含义，例如，在转移 lamports 时，可以将第一个帐户定义为源帐户，第二个帐户定义为目的帐户。

除了报表和数据之外，```AccountInfo```结构的成员是只读的。两者都可以由程序根据运行时执行策略([runtime enforcement policy](https://docs.solana.com/developing/programming-model/accounts#policy))进行修改。这两个成员都受到Rust ```RefCell```构造的保护，因此必须借用它们来读取或写入它们。这样做的原因是它们都指向原始输入字节数组，但是帐户切片中可能有多个条目指向同一个帐户。使用```RefCell```确保程序不会意外地通过多个```AccountInfo```结构对相同的底层数据执行重叠的读/写。如果一个程序实现了自己的反序列化功能，那么应该注意适当地处理重复的帐户。

指令数据是被处理指令数据的通用字节数组

### 堆
Rust 程序通过自定义 [global_allocator](https://github.com/solana-labs/solana/blob/8330123861a719cd7a79af0544617896e7f00ce3/sdk/program/src/entrypoint.rs#L50) 来直接实现堆.

程序可以根据特定的需要实现自己的global_allocator。 查看[custom heap example](https://docs.solana.com/developing/on-chain-programs/developing-rust#examples)获取更多信息

### 限制
链上Rust程序支持大多数Rust的libstd、libcore和liballoc，以及许多第三方crate。

由于这些程序运行在资源受限的单线程环境中，并且必须是确定性的，因此有一些限制:
- No access to

	- rand 
	- std::fs
	- std::net
	- std::os
	- std::future
	- std::process
	- std::sync
	- std::task
	- std::thread
	- std::time 
- 限制使用

	- std::hash
	- std::os
- 在循环和调用深度方面，Bincode在计算上都非常昂贵，应该避免使用
- 应该避免字符串格式化，因为这在计算上也很昂贵。
- 不支持println!,print!，应该使用Solana日志帮助程序。
- 运行时对程序在处理一条指令的过程中可以执行的指令数进行限制。更多信息请参见[computation budget](https://docs.solana.com/developing/programming-model/runtime#compute-budget)

### Depending on Rand 
程序被限制为确定性运行，因此随机数不可用。有些时候程序所依赖的crate依赖于它自己的 rand，即使这个程序没有使用任何random number相关的函数。如果一个程序依赖于rand，编译将会失败，因为 Solana 没有对的 ```get-random``` 的支持。这个错误通常看起来像这样:
```
error: target is not supported, for more information see: https://docs.rs/getrandom/#unsupported-targets
   --> /Users/jack/.cargo/registry/src/github.com-1ecc6299db9ec823/getrandom-0.1.14/src/lib.rs:257:9
    |
257 | /         compile_error!("\
258 | |             target is not supported, for more information see: \
259 | |             https://docs.rs/getrandom/#unsupported-targets\
260 | |         ");
    | |___________^
```
要解决这个依赖问题，将以下依赖项添加到程序的Cargo.toml中:
```
getrandom = { version = "0.1.14", features = ["dummy"] }
```

### 日志
Rust 的println !宏在计算上开销很大，不受支持。作为替代，提供宏msg!
msg! 有两种格式
```
msg!("A string");
```
或者
```
msg!(0_64, 1_64, 2_64, 3_64, 4_64);
```
两种格式都会输出到程序日志中，如果程序希望它们可以模仿println!通过使用格式!:
```
msg!("Some variable: {:?}", variable);
```
调试部分(https://docs.solana.com/developing/on-chain-programs/debugging#logging)有更多关于使用程序日志的信息，Rust示例([rust examples](https://docs.solana.com/developing/on-chain-programs/developing-rust#examples))包含一个日志示例。

### Panicking
Rust的 panic!, assert!,和 internal panic 结果都会打印到程序的日志中。
类似下面的打印
```
INFO  solana_runtime::message_processor] Finalized account CGLhHSuWsp1gT4B7MY2KACqp9RUwQRhcUFfVSuxpSajZ
INFO  solana_runtime::message_processor] Call BPF program CGLhHSuWsp1gT4B7MY2KACqp9RUwQRhcUFfVSuxpSajZ
INFO  solana_runtime::message_processor] Program log: Panicked at: 'assertion failed: `(left == right)`
      left: `1`,
     right: `2`', rust/panic/src/lib.rs:22:5
INFO  solana_runtime::message_processor] BPF program consumed 5453 of 200000 units
INFO  solana_runtime::message_processor] BPF program CGLhHSuWsp1gT4B7MY2KACqp9RUwQRhcUFfVSuxpSajZ failed: BPF program panicked
```

### 自定义panic handler
程序可以实现自己的panic来重写默认的panic。

首先在Cargo.toml 里面定义 custom-panic 特性
```
[features]
default = ["custom-panic"]
custom-panic = []
```
然后提供自定义实现
```
#[cfg(all(feature = "custom-panic", target_arch = "bpf"))]
#[no_mangle]
fn custom_panic(info: &core::panic::PanicInfo<'_>) {
    solana_program::msg!("program custom panic enabled");
    solana_program::msg!("{}", info);
}
```
在上面的代码片段中，显示了默认的实现，但是开发人员可以用更适合他们需要的东西来替换它。

默认情况下支持完整的恐慌消息的一个副作用是，程序需要将更多的Rust的libstd实现引入程序的共享对象。典型的程序已经使用了相当多的libstd，可能不会注意到共享对象大小的增加。但是，对于通过避免libstd显式地试图变得非常小的程序可能会产生很大的影响(~25kb)。为了消除这种影响，程序可以用一个空的实现提供它们自己的custom panic handler。
```
#[cfg(all(feature = "custom-panic", target_arch = "bpf"))]
#[no_mangle]
fn custom_panic(info: &core::panic::PanicInfo<'_>) {
    // Do nothing to save space
}
```

### 计算预算（Compute Budget）
使用系统调用sol_log_compute_units()来记录一条消息，其中包含在暂停执行之前程序可能消耗的剩余计算单元数

查看[compute budget](https://docs.solana.com/developing/programming-model/runtime#compute-budget)了解更多

### ELF Dump
BPF共享对象的内部内容可以转储到一个文本文件中，以便更深入地了解程序的组成以及它在运行时可能在做什么。转储将包含ELF信息以及所有符号和实现它们的指令的列表。一些BPF加载器的错误日志消息将引用发生错误的特定指令号。可以在ELF转储中查找这些引用，以识别出错的指令及其上下文。

创建dump 文件
```
$ cd <program directory>
$ cargo build-bpf --dump
```

### 例子
The [Solana Program Library github](https://github.com/solana-labs/solana-program-library/tree/master/examples/rust) repo contains a collection of Rust examples.
