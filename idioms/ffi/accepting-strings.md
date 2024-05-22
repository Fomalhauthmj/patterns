# 接受字符串

## 描述

当FFI通过指针接受字符串时，应该遵循两个原则：

1. 保持外部字符串是“借用”的，而不是直接复制它们。
2. 尽量减少从C风格字符串转换到原生Rust字符串时涉及的复杂性和`unsafe`代码量。

## 动机

C语言中使用的字符串与Rust语言中使用的字符串有不同的行为：

- C语言的字符串是以`NULL`(`'\0'`)终止的，而Rust语言的字符串会存储其长度。
- C语言的字符串可以包含任何任意的非零字节，而Rust的字符串必须是UTF-8。
- C语言的字符串使用`unsafe`的指针操作来访问和操作，而与Rust字符串的交互是通过安全方法进行的。

Rust标准库提供了与Rust的`String`和`&str`相对应的C语言等价表示，称为`CString`和`&CStr`，这使得我们可以避免在C语言字符串和Rust字符串之间转换的复杂性和`unsafe`代码。

`&CStr`类型还允许我们使用借用数据，这意味着在Rust和C之间传递字符串是一个零成本的操作。

## 代码示例

```rust,ignore
pub mod unsafe_module {

    // other module content

    /// Log a message at the specified level.
    ///
    /// # Safety
    ///
    /// It is the caller's guarantee to ensure `msg`:
    ///
    /// - is not a null pointer
    /// - points to valid, initialized data
    /// - points to memory ending in a null byte
    /// - won't be mutated for the duration of this function call
    #[no_mangle]
    pub unsafe extern "C" fn mylib_log(
        msg: *const libc::c_char,
        level: libc::c_int
    ) {
        let level: crate::LogLevel = match level { /* ... */ };

        // SAFETY: The caller has already guaranteed this is okay (see the
        // `# Safety` section of the doc-comment).
        let msg_str: &str = match std::ffi::CStr::from_ptr(msg).to_str() {
            Ok(s) => s,
            Err(e) => {
                crate::log_error("FFI string conversion failed");
                return;
            }
        };

        crate::log(msg_str, level);
    }
}
```

## 优势

这个例子的编写是为了确保：

1. `unsafe`块尽可能小。
2. 具有“未跟踪”的生命周期的指针成为“跟踪”的共享引用。

考虑一个替代方案，即实际复制字符串：

```rust,ignore
pub mod unsafe_module {

    // other module content

    pub extern "C" fn mylib_log(msg: *const libc::c_char, level: libc::c_int) {
        // DO NOT USE THIS CODE.
        // IT IS UGLY, VERBOSE, AND CONTAINS A SUBTLE BUG.

        let level: crate::LogLevel = match level { /* ... */ };

        let msg_len = unsafe { /* SAFETY: strlen is what it is, I guess? */
            libc::strlen(msg)
        };

        let mut msg_data = Vec::with_capacity(msg_len + 1);

        let msg_cstr: std::ffi::CString = unsafe {
            // SAFETY: copying from a foreign pointer expected to live
            // for the entire stack frame into owned memory
            std::ptr::copy_nonoverlapping(msg, msg_data.as_mut(), msg_len);

            msg_data.set_len(msg_len + 1);

            std::ffi::CString::from_vec_with_nul(msg_data).unwrap()
        }

        let msg_str: String = unsafe {
            match msg_cstr.into_string() {
                Ok(s) => s,
                Err(e) => {
                    crate::log_error("FFI string conversion failed");
                    return;
                }
            }
        };

        crate::log(&msg_str, level);
    }
}
```

这个版本的代码在两个方面比原版逊色：

1. 有更多的`unsafe`代码，更重要的是，它必须坚持更多的不变量。
2. 由于需要大量的算术，这个版本有一个错误，会导致Rust的`undefined behaviour`。

这里的错误是一个简单的指针运算错误：字符串所有的`msg_len`字节被复制了。
但是，结尾的`NUL`终止符没有被复制。

然后，Vector的大小被*设置*为*zero padded string*的长度——而不是*调整大小*到它，即可能会在最后添加一个零。
结果是，Vector中的最后一个字节是未初始化的内存。
当`CString`在块的底部被创建时，它对Vector的读取将导致`undefined behaviour`！

像许多这样的问题一样，这将是一个很难追踪的问题。
有时它会因为字符串不是`UTF-8`而panic，有时它会在字符串的末尾放一个奇怪的字符，有时它会完全崩溃。

## 劣势

没有？

> Latest commit 606bcff on 26 Feb 2021
