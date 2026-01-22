---
title: C2Rust
summary: C自动转换Rust
date: 2026-01-22
authors:
  - qwertlooker
tags:
  - Rust
  - c&c++
---

# 论文
[c自动转换Rust](https://cacm.acm.org/research/automatically-translating-c-to-rust/)

# 相关项目
 [CRustS](https://doi.org/10.1145/3510454.3528640) （采用基于语法规则的翻译以降低 unsafe 比例）
 [CROWN](https://doi.org/10.1007/978-3-031-37709-9_22)（利用 Rust 编译器将C的裸指针转换为Rust的Boxed引用）
 [RustMap](https://link.springer.com/chapter/10.1007/978-3-032-00828-2_16)（使用 GPT-4 大模型将 bzip2 等中型C项目重写为 Rust）。
[](https://eurorust.eu/2025/talks/rewrite-optimize-repeat/)

# rust原理相关
RediSearch. "Compare the code branches in C and Rust". November, 2025.
Nora Trieb. "How Rust Compiles", talk at Euro Rust 2025.
Jana Donzelmann. "What actually are attributes?", talk at Euro Rust 2025.
