# Rust-toolchain

## 工具链管理
rustup是Rust工具链的installer，安装rust，管理rust工具链都要借助这个工具来做。

### 安装nightly工具链
stable工具链最常用，但有时需要用到一些nightly的特性 or 测试一些新特性的时候，需要安装nightly工具链：
```bash
rustup install nightly
```

### 工具链切换
一般来说，默认工具链设置为stable工具链，但如果需要讲默认工具链设置为`nightly`，可以使用`default set` 子命令；

更常见的需求是，在当前项目中使用 nightly 工具链，但保持全局设置不变，这时可以在当前project中使用`ovveride set`子命令：
```bash
rustup default set nightly

rustup override set nightly
```

### Others
```bash
# 列出active工具链的所有可用target
rustup target list

# 安装 Android target（交叉编译如此简单）
rustup target add arm-linux-androideabi

# 运行为nightly工具链配置的 shell
rustup run nightly bash

# 显示当前目录将使用哪个工具链
rustup show
```
## Cargo 换源

打开`~/.cargo/`目录，并在该目录下新建config文件：

```bash
cd ~/.cargo/
touch config
```

编辑 config 内容如下（设置tuna源）：

```text
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'tuna'

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
```
