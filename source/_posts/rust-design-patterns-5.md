---
title: Rust Design Pattern 讀後整理 - Anti-patterns
date: 2021-11-14 13:01:59
tags: 
  - rust
  - design pattern
---

# 4. Anti-patterns
## 4.1. Clone to satisfy the borrow checker
borrow checker的用意是避免開發者寫出unsafe的程式碼，不應該為了應付compiler而使用clone去滿足borrow check。
<!-- more -->

```rust
let mut x = 5;
let y = &mut (x.clone());
*y += 1;
println!("{}", x); // 如果沒有x.clone()，這邊會被擋下
```

## 4.2. #![deny(warnings)]
立意良善的crate作者會確保他們的程式編譯會沒有警告產生。所以有些人會在crate root的地方加上:
```rust
#![deny(warnings)]
```

這樣寫很簡單而且會讓warning產生時中止編譯。
但是這種作法會讓crate失去Rust提供的穩定性，因為有時候新功能出現或舊的錯誤功能會透過warning產生提示。如果讓這種warning停止了編譯

### 替代方案
  1. 將build setting跟程式碼分離，透過command line設定deny
    `RUSTFLAGS="-D warnings" cargo build`
  2. 指定想要deny的warning清單，以下兩組都是合適的deny清單:
    ```rust
    #[deny(bad-style,
       const-err,
       dead-code,
       improper-ctypes,
       non-shorthand-field-patterns,
       no-mangle-generic-items,
       overflowing-literals,
       path-statements ,
       patterns-in-fns-without-body,
       private-in-public,
       unconditional-recursion,
       unused,
       unused-allocation,
       unused-comparisons,
       unused-parens,
       while-true)]
    #[deny(missing-debug-implementations,
       missing-docs,
       trivial-casts,
       trivial-numeric-casts,
       unused-extern-crates,
       unused-import-braces,
       unused-qualifications,
       unused-results)]
    ```

## 4.3. Deref Polymorphism
濫用`Deref` trait來模擬struct之間的繼承，進而達到重用方法的效用。

假設我們想要模仿OO語言如Java的這個情況:
```java
class Foo {
    void m() { ... }
}
class Bar extends Foo {}
public static void main(String[] args) {
    Bar b = new Bar();
    b.m();
}
```

在rust可以透過這種方法達成:
```rust
use std::ops::Deref;
struct Foo {}
impl Foo {
    fn m(&self) {
        //..
    }
}

struct Bar {
    f: Foo,
}
impl Deref for Bar {
    type Target = Foo;
    fn deref(&self) -> &Foo {
        &self.f
    }
}

fn main() {
    let b = Bar { f: Foo {} };
    b.m();
}
```

為了要讓這方法成功，實作`Deref`而讓`Bar`的dereference回傳`Foo`會是很怪異的事情。在Rust沒有struct的繼承，一般是使用組合來代替。