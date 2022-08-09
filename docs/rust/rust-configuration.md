---
title: Rust工具链+Cargo换源
date: 2021-12-10 14:28:38
tags: ['Rust']
categories: ['Rust']
description: "Rust 切换工具链+cargo 换源..."
cover: "https://github.com/cat538/images-auto/raw/main/img/rustlogo.jpg"
---

## 切换工具链

```shell
rustup default set nightly

rustup override set nightly
```

## 换源

打开~/.cargo/目录，并在该目录下新建config文件：

```shell
cd ~/.cargo/
touch config
```

编辑 config 内容如下：

```text
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'tuna'

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
```

把cargo更换成清华源
