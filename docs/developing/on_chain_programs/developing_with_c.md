## C开发
### 项目布局
C的项目结构类似下面
```
/src/<program name>
/makefile
```
makefile 中要包含如下内容
```
OUT_DIR := <path to place to resulting shared object>
include ~/.local/share/solana/install/active_release/bin/sdk/bpf/c/bpf.mk
```
bpf-sdk可能不在上面指定的确切位置，但如果你按照How to Build设置你的环境，那么它应该在。

可以参考 [helloword](https://github.com/solana-labs/example-helloworld/tree/master/src/program-c) 的例子

### 如何构建
先安装环境

- 安装最新稳定版本的Rust [https://rustup.rs/](https://rustup.rs/)
- 安装最新版本的Solana命令行工具 [https://docs.solana.com/cli/install-solana-cli-tools](https://docs.solana.com/cli/install-solana-cli-tools)

然后使用make 编译
```
make -C <program directory>
```
