---
title: Rust Design Pattern 讀後整理 - Idioms(上)
date: 2021-11-07 21:16:15
tags: rust
---

# 2. Idioms
## 2.1. Use borrowed types for arguments
參數優先使用borrowed type，而非borrowing the owned type，避免多一層間接的使用，也更能被廣泛使用，像是： 
  - 使用`&str`而非`&String`
  - 使用`&[T]` 而非 `&Vec<T>`
  - 使用`&T` 而非 `&Box<T>`
<!-- more -->

例如下方的`three_vowels2`可以更好的支援String以及str兩種型態

```rust
fn three_vowels(word: &String) -> bool {
  ...
}

fn three_vowels2(word: &str) -> bool {
  ...
}

fn main() {
  let s = "str".to_string();
  three_vowels(&s);
  // not working three_vowels("string");
  three_vowels2(&s);
  three_vowels2("string");
}
```

## 2.2. Concatenating strings with format!
使用`format!`來串接字串，而不是使用`push`, `push_str`, `+`

```rust
fn say_hello(name: &str) -> String {
    // We could construct the result string manually.
    // let mut result = "Hello ".to_owned();
    // result.push_str(name);
    // result.push('!');
    // result

    // But using format! is better.
    format!("Hello {}!", name)
}
```

  - 優點: 方便、可讀性高
  - 缺點: `push`的效能比較好

## 2.3. Constructors
Rust並沒有支援建構子，建議使用`new` function來實作。

```rust
struct MyObj {
  n: i32,
}
impl MyObj {
  fn new() -> Self {
    Self {
      n: 0
    }
  }
}
```

## 2.4. The Default Trait
- 實作Default Trait可以定義struct內各欄位的預設值
- 實作Default Trait可以讓你的struct與其他的container和generic types合作的更好(如: Option::unwrap_or_default())。
- 若是該struct所有的欄位都有實作`Default`，則可透過attribute讓struct直接支援default

```rust
#[derive(Default)]
struct SomeOptions {
    foo: i32,
    bar: i32,
}
fn main() {
    let o = SomeOptions::default();
    assert_eq!(o.foo, 0);
    assert_eq!(o.bar, 0);

    let o2 = SomeOptions {
      foo: 1,
      ..Default::default()
    };
    assert_eq!(o2.foo, 1);
    assert_eq!(o2.bar, 0);
}
```

## 2.5. Collections are smart pointers
使用`Deref` trait 讓容器可以像smart pointer提供data的owning以及borrowed view
- 優點: 大多數的method可以實作在borrowed view即可，因為能被owing view隱式轉換

## 2.6. Finalisation in Destructors
Rust沒有提供`finally`區塊的語法，而是以物件解構的方式代替執行，方法是實作`Drop` trait。

## 2.7. mem::{take(_), replace(_)} to keep owned values in changed enums
做變數替換時，為了滿足borrower checker又不濫用`clone()`，可使用`mem::take()`或`mem::replace()`

```rust
use std::mem;

enum MyEnum {
    A { name: String, x: u8 },
    B { name: String }
}

fn a_to_b(e: &mut MyEnum) {
    if let MyEnum::A { name, x: 0 } = e {
        *e = MyEnum::B { name: mem::take(name) }
    }
}
```