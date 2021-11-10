---
title: Rust Design Pattern 讀後整理 - Idioms(下)
date: 2021-11-08 20:36:02
tags: rust
---

# 2. Idioms
## 2.8. On-Stack Dynamic Dispatch
透過提前宣告多個不同型態的變數來達成On-Stack的動態分配
<!-- more -->

### 範例
```rust
use std::io;
use std::fs;
// 事先宣告readable可能變成的型態
let (mut stdin_read, mut file_read);

let readable: &mut dyn io::Read = if arg == "-" {
    stdin_read = io::stdin();
    &mut stdin_read
} else {
    file_read = fs::File::open(arg)?;
    &mut file_read
};
```

- 優點: 不必在heap分配空間
- 缺點: 程式碼比起使用`Box`來得長
```rust
let readable: Box<dyn io::Read> = if arg == "-" {
    Box::new(io::stdin())
} else {
    Box::new(fs::File::open(arg)?)
};
```

## 2.9. Foreign function interface (FFI)
### Error Handling in FFI
在FFI處理錯誤時，讓Rust與其他語言更好銜接的方法:
1. 利用Enum對應error code類型的回傳值時，支援從error code轉換
```rust
enum DatabaseError {
    IsReadOnly = 1, // user attempted a write operation
    IOError = 2, // user should read the C errno() for what it was
    FileCorrupted = 3, // user should run a repair tool to recover it
}

impl From<DatabaseError> for libc::c_int {
    fn from(e: DatabaseError) -> libc::c_int {
        (e as i8).into()
    }
}
```
2. 利用structed Enum時，配上string 提供detail error message
```rust
pub mod errors {
    enum DatabaseError {
        IsReadOnly,
        IOError(std::io::Error),
        FileCorrupted(String), // message describing the issue
    }

    impl From<DatabaseError> for libc::c_int {
      ...
    }
}

pub mod c_api {
    use super::errors::DatabaseError;

    #[no_mangle]
    pub extern "C" fn db_error_description(
        e: *const DatabaseError
        ) -> *mut libc::c_char {

        let error: &DatabaseError = unsafe {
            // SAFETY: pointer lifetime is greater than the current stack frame
            &*e
        };

        let error_str: String = match error {
            DatabaseError::IsReadOnly => {
                format!("cannot write to read-only database");
            }
            DatabaseError::IOError(e) => {
                format!("I/O Error: {}", e);
            }
            DatabaseError::FileCorrupted(s) => {
                format!("File corrupted, run repair: {}", &s);
            }
        };

        let c_error = unsafe {
            // 省略： 此部份將error_str複製到宣告好的buffer並加上NUL結尾
        };

        c_error
    }
}
```

3. 自定義Error Type要額外附加一個C語言結構的exposure
```rust
struct ParseError {
    expected: char,
    line: u32,
    ch: u16
}

impl ParseError { /* ... */ }

/* Create a second version which is exposed as a C structure */
#[repr(C)]
pub struct parse_error {
    pub expected: libc::c_char,
    pub line: u32,
    pub ch: u16
}

impl From<ParseError> for parse_error {
    fn from(e: ParseError) -> parse_error {
        let ParseError { expected, line, ch } = e;
        parse_error { expected, line, ch }
    }
}
```

### Accepting Strings
從FFI接收string時，保持兩個原則:
  1. 將string維持"borrowed"狀態，而不是直接複製
  2. 從C-style string來時，減少複雜度以及`unsafe`區塊

### Passing Strings
傳遞string進FFI時，保持四個原則:
  1. 將string的lifetime盡可能拉長
  2. 減少轉換用的`unsafe`區塊
  3. 當C程式端可以修改string時，使用`Vec`而不是`CString`
  4. 除非API需要，所有權不應移交給callee

## 2.10. Iterating over an Option
`Option`有實作`IntoIterator` trait，可以被視為一個含有0或1個元素的容器。

### 範例
```rust
let turing = Some("Turing");
let mut logicians = vec!["Curry", "Kleene", "Markov"];

logicians.extend(turing);

// 等同於
if let Some(turing_inner) = turing {
    logicians.push(turing_inner);
}
```

## 2.11. Pass Variables to Closure
closure預設將變數透過借用傳入，或是加上`move`改為move傳入。可以透過變數重新綁定的方法，針對個別變數做不同的行為。

### 範例
```rust
use std::rc::Rc;

let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);

let num2_cloned = num2.clone();
let num3_borrowed = num3.as_ref();
let closure = move || {
    *num1 + *num2_cloned + *num3_borrowed;
};
```

## 2.12. #[non_exhaustive] and private fields for extensibility
想要替library的struct, enum增加public欄位卻又不想破壞向下相容性時，Rust提供兩個解法: 
### 使用`#[non_exhaustive]`
```rust
mod a {
    #[non_exhaustive]
    pub struct S {
        pub foo: i32,
    }
    
    #[non_exhaustive]
    pub enum AdmitMoreVariants {
        VariantA,
        VariantB,
    }
}

fn print_matched_variants(s: a::S) {
    // 因為S被加上`#[non_exhaustive]`，所以必須加上 ..
    let a::S { foo: _, ..} = s;
    
    let some_enum = a::AdmitMoreVariants::VariantA;
    match some_enum {
        a::AdmitMoreVariants::VariantA => println!("it's an A"),
        a::AdmitMoreVariants::VariantB => println!("it's a b"),
        // 因為`#[non_exhaustive]`，這邊也必須處理未來新增的情況
        _ => println!("it's a new variant")
    }
}
```

### 增加private欄位
`#[non_exhaustive]`只能用於跨crate，在同crate內就必須使用private欄位。

```rust
pub struct S {
    pub a: i32,
    // 因為private `b`的存在，要match時就要配上`..`
    _b: ()
}
```

### 缺點
這種作法會導致程式碼不符合人體工學，而且還要強迫使用者處理未知情況，多數情形下都是無意義的處理。

## 2.13. Easy doc initialization
在撰寫文件範例時，如果一個參數需要費點功夫才能初始化，可以用一個helper function的形式將範例包裝起來。

缺點是被包在function內的範例無法被測試。

### 範例
下方Example內的`request`原本要透過一系列的步驟才能生出來，透過將範例包進一個`call_send` function，並將request當作參數帶進來即可省略初始化。

```rust
struct Connection {
    name: String,
    stream: TcpStream,
}

impl Connection {
    /// Sends a request over the connection.
    /// 
    /// # Example
    /// ```
    /// # fn call_send(connection: Connection, request: Request) {
    /// let response = connection.send_request(request);
    /// assert!(response.is_ok());
    /// # }
    /// ```
    fn send_request(&self, request: Request) {
        // ...
    }
}
```

## 2.14. Temporary mutability
有些時候資料可能需要先被準備以及處理過，但在那之後資料就不會再被進行修改，所以可以特別將資料從mutable改為immutable。

方法有兩種:
  - nested block: 將資料在block內進行資料準備與處理，然後再塞進data
  ```rust
  let data = {
      let mut data = get_vec();
      data.sort();
      data
  };
  ```

  - variable rebinding: 利用變數可以重新綁定的語法達成。
  ```rust
  let mut data = get_vec();
  data.sort();
  let data = data;
  ```
