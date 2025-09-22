+++
title = "Rust 在 Linux Kernel 中 alloc 的链路分析"
date = 2025-09-02
[taxonomies]
categories = ["Rust For Linux"]
tags = ["rust", "allocator", "kmalloc", "krealloc", "kernel", "rust-for-linux"]
+++

本文尝试从 **Rust 层调用** → **FFI bindings** → **C helper** → **内核 API 宏/函数** → **实际分配器**
一步步梳理 Rust 在 Linux 内核里的内存分配调用链。
以 `KVec::push` 触发扩容为例，串联 `krealloc` 的全过程。

<!--more-->

## 背景

在用户态，Rust 的分配接口最终会落到 `malloc` / `realloc` 等系统调用。
但在 **Linux 内核** 中，我们不能直接用 libc，而必须调用内核自己的分配器接口：`kmalloc`、`krealloc`、`kfree` 等。

Rust-for-Linux 为此定义了 **安全抽象层**：

- `Allocator` trait：Rust 世界的分配器接口
- `Kmalloc`：面向内核的具体实现
- `KBox` / `KVec`：高层数据结构的封装，背后依赖分配器

## 从 `KVec::push` 说起

**文件路径：** [`rust/kernel/alloc/kvec.rs`](https://github.com/Rust-for-Linux/linux/blob/1b237f190eb3d36f52dffe07a40b5eb210280e00/rust/kernel/alloc/kvec.rs#L312-L318)

```rust

    pub fn push(&mut self, v: T, flags: Flags) -> Result<(), AllocError> {
        self.reserve(1, flags)?;
        // SAFETY: The call to `reserve` was successful, so the capacity is at least one greater
        // than the length.
        unsafe { self.push_within_capacity_unchecked(v) };
        Ok(())
    }
```

**执行流程：**

1. `reserve(1, flags)` 检查容量，不够则触发扩容
2. 扩容路径进入 `Kmalloc::realloc`，核心就是走到 C 侧的 `krealloc`
3. `push_within_capacity_unchecked` 只是写入数据，不涉及分配

## Rust 侧：Allocator → FFI

**文件路径：** [`rust/kernel/alloc/allocator.rs`](https://github.com/Rust-for-Linux/linux/blob/1b237f190eb3d36f52dffe07a40b5eb210280e00/rust/kernel/alloc/allocator.rs#L130-L143)

```rust

unsafe impl Allocator for Kmalloc {
    #[inline]
    unsafe fn realloc(
        ptr: Option<NonNull<u8>>,
        layout: Layout,
        old_layout: Layout,
        flags: Flags,
    ) -> Result<NonNull<[u8]>, AllocError> {
        let layout = Kmalloc::aligned_layout(layout);

        // SAFETY: `ReallocFunc::call` has the same safety requirements as `Allocator::realloc`.
        unsafe { ReallocFunc::KREALLOC.call(ptr, layout, old_layout, flags) }
    }
}

// 自动生成
extern "C" {
    #[must_use]
    #[link_name="rust_helper_krealloc"]
    pub fn krealloc(
        objp: *const ffi::c_void,
        new_size: usize,
        flags: gfp_t,
    ) -> *mut ffi::c_void;
}
```

**关键点：**

- `extern "C"`：告诉 Rust 这个函数在 C 世界里实现
- `#[link_name="rust_helper_krealloc"]`：实际链接的符号名

## C 侧 Helper

Rust 无法直接绑定 `krealloc` 宏（它是个宏/inline，不是符号）。所以内核提供一个 helper：

**文件路径：** [`rust/helpers/slab.c`](https://github.com/Rust-for-Linux/linux/blob/1b237f190eb3d36f52dffe07a40b5eb210280e00/rust/helpers/slab.c#L5-L9)

```c

void * __must_check __realloc_size(2)
rust_helper_krealloc(const void *objp, size_t new_size, gfp_t flags)
{
	return krealloc(objp, new_size, flags);
}
```

这里 `krealloc(...)` 宏会在 C 里展开为：

**文件路径：** [`include/linux/slab.h`](https://github.com/Rust-for-Linux/linux/blob/1b237f190eb3d36f52dffe07a40b5eb210280e00/include/linux/slab.h#L468C1-L470C67)

```c

void * __must_check krealloc_noprof(const void *objp, size_t new_size,
				    gfp_t flags) __realloc_size(2);
#define krealloc(...)				alloc_hooks(krealloc_noprof(__VA_ARGS__))
```

## 内核侧：宏 → 真正分配器

1. **`krealloc` 宏**：加上 `alloc_hooks(...)`，触发 KASAN、KFENCE、memcg、trace 等调试与统计逻辑

2. **`krealloc_noprof` 函数**：这是实际分配函数。行为：
   - 若 `objp == NULL` → 相当于 `kmalloc(new_size, flags)`
   - 若能原地扩展 → 直接扩大
   - 否则 → 新分配 + memcpy 拷贝旧数据 + 释放旧块

3. **底层分配器**：最终由 SLAB / SLUB / SLOB 或 vmalloc 等实现，取决于大小、flags 和内核配置

## 小结

### 为什么要 helper？

因为很多内核 API 是宏/内联函数，没有符号；Rust 必须绑定一个"稳定符号"，所以需要 C helper。

### bindings vs helper

- **bindings**：Rust 端的 FFI 声明
- **helper**：C 端的薄封装，确保能链接，并走对的内核路径

### 落地路径

```
Rust 抽象（KVec/KBox）
    ↓
分配器 trait (Allocator)
    ↓
FFI (extern "C")
    ↓
C helper (rust_helper_*)
    ↓
内核 API 宏/函数
    ↓
实际的 slab/vmalloc 分配器
```
