---
title: Rust Design Pattern 讀後整理 - Design Patterns(下)
date: 2021-11-13 12:47:26
tags: 
  - rust
  - design pattern
---

# 3. Design Patterns
## 3.3. Structural
了解Entity之間的關係來簡化設計。
<!-- more -->
### 3.3.1. Compose structs together for better borrowing
有些時候較大的struct使用上會受到borrow checker的干擾，雖然欄位可以被單獨借用，但有時候也會變成整個struct被限制。解決的方法是將struct拆成幾個較小的strcut，然後再將他們組合起來成原來的struct，這樣一來每個小struct都可以被個別借用，增進彈性。

```rust
struct A {
    f1: u32,
    f2: u32,
    f3: u32,
}
fn foo(a: &mut A) -> &u32 { &a.f2 }
fn bar(a: &mut A) -> u32 { a.f1 + a.f3 }

fn baz(a: &mut A) {
    // 這邊借用了整個a
    let x = foo(a);
    // Borrow checker error: 因為a被x拿走了，這邊不能再借一次
    let y = bar(a);
    println!("{}", x);
}
```

將`struct A`拆成`struct B`和`struct C`，然後就可以單獨借用B, C
```rust
struct A {
    b: B,
    c: C,
}
struct B {
    f2: u32,
}
struct C {
    f1: u32,
    f3: u32,
}
fn foo(b: &mut B) -> &u32 { &b.f2 }
fn bar(c: &mut C) -> u32 { c.f1 + c.f3 }
fn baz(a: &mut A) {
    let x = foo(&mut a.b);
    let y = bar(&mut a.c);
    println!("{}", x);
}
```

### 3.3.2. Prefer Small Crates
偏好只將一件事做好的小型crate。

#### 優點
  * 小型crate較容易讓人理解，也能讓程式碼較具模組化
  * 更容易讓不同專案重複使用
  * Rust編譯的最小單元是crate，拆分專案成不同的crate有助於平行編譯

#### 缺點
  * 當專案依賴不同衝突版本時，可能導致"dependency hell"
  * crates.io上的專案不一定品質都好
  * 兩個小型crate在最佳化上可能不如一個大的，因為compiler預設無法做link-time optimization(LTO)

### 3.3.3. Contain unsafety in small modules
如果有`unsafe`的程式碼，可以將他們包裝成盡可能小的模組，再配上一個安全的介面。

#### 優點
  * 可以將`unsafe`的程式碼限制
  * 寫外圍模組會比較簡單，因為可以依賴較有保證的內部模組

#### 缺點
  * 有時候不易找到適合的介面
  * 這種抽象可能導致效率較差

## 3.4. Foreign function interface (FFI)
### 3.4.1. Object-Based APIs
當設計輸出給其他語言的API時，有幾點設計原則與一般的Rust API不同: 
  1. 所有封裝的類型必須保持Rust擁有，由user管理並且不透明
  2. 所有Transactional的資料類型應該由user擁有，並且透明
  3. 所有library的行為應是function作用在封裝的類型上
  4. 所有library的行為應被封裝成不基於struct上的類型，而是基於provenance/lifetime

#### 範例: C語言 object-based API
- `DBM`即為被封裝的類型，對user來說是不透明的，只能透過`dbm_open`來建立。並且透過`dbm_close`來管理生命週期。
- `datum`屬於Transactional的資料
```c
struct DBM;
typedef struct { void *dptr, size_t dsize } datum;

int     dbm_clearerr(DBM *);
void    dbm_close(DBM *);
int     dbm_delete(DBM *, datum);
int     dbm_error(DBM *);
datum   dbm_fetch(DBM *, datum);
datum   dbm_firstkey(DBM *);
datum   dbm_nextkey(DBM *);
DBM    *dbm_open(const char *, int, mode_t);
int     dbm_store(DBM *, datum, datum, int);
```

### 3.4.2. Type Consolidation into Wrappers
這模式能讓人優雅的處理多個關聯的類型，同時最小化memory unsafey的區域。
Rust輸出給其他語言時一般會轉為pointer，也代表著由user管理生命週期。為了降低風險，會使用"consolidated wrapper"，將可能的操作合併到一個wrapper type。

```rust
struct MySetWrapper {
    myset: MySet,
    iter_next: usize,
}
impl MySetWrapper {
    pub fn first_key(&mut self) -> Option<&Key> {
        self.iter_next = 0;
        self.next_key()
    }
    pub fn next_key(&mut self) -> Option<&Key> {
        if let Some(next) = self.myset.keys().nth(self.iter_next) {
            self.iter_next += 1;
            Some(next)
        } else {
            None
        }
    }
}
```