---
title: Rust Design Pattern 讀後整理 - Design Patterns
date: 2021-11-11 09:06:20
tags: 
  - rust
  - design pattern
---

# 3. Design Patterns
設計模式為一般常見問題的解決方案。一般也會跟使用的語言習習相關，因為每個語言的特性和擁有的功能不同，一個語言適用的模式在其他語言不一定適用，因此也很適合從設計模式來描述一個語言的文化。

## YAGNI
`You Aren't Going to Need It` 是一套很重要的軟體開發原則，我個人認為比其他原則都還優先，思考是否真的要套用設計模式也是很重要的一環，不要為了會用而用，進而造成過度設計。
<!-- more -->
## 3.1. Behavioural Patterns
用來辨別物件之間常見的溝通模式，透過這些模式可以增加溝通的彈性。

### 3.1.1. Command
基本概念是將各個動作分成獨自的object並像參數一樣進行傳遞。

#### 動機
有一系列的動作事先被包裝起來，並要以某個順序執行。在某個事件觸發後才執行。

#### 方法
 * 透過`trait` objects
  ```rust
  pub trait Migration {
      fn execute(&self) -> &str;
  }
  pub struct CreateTable;
  impl Migration for CreateTable {
      fn execute(&self) -> &str {"create table"}
  }
  pub struct AddField;
  impl Migration for AddField {
      fn execute(&self) -> &str {"add field"}
  }
  struct Schema {
      commands: Vec<Box<dyn Migration>>,
  }
  impl Schema {
      // 省略new & push command
      // ...
      fn execute(&self) -> Vec<&str> {
          self.commands.iter().map(|cmd| cmd.execute()).collect()
      }
  }
  ```
 * 透過function pointers
  ```rust
  type FnPtr = fn() -> String;
  struct Command {
      execute: FnPtr,
  }
  struct Schema {
      commands: Vec<Command>,
  }
  impl Schema {
      // 省略new & push command
      // ...
      fn execute(&self) -> Vec<String> {
          self.commands.iter().map(|cmd| (cmd.execute)()).collect()
      }
  }
  fn add_field() -> String {
      "add field".to_string()
  }
  ```
 * 透過 `Fn` trait objects
  ```rust
  type Migration<'a> = Box<dyn Fn() -> &'a str>;
  struct Schema<'a> {
      executes: Vec<Migration<'a>>,
  }
  impl<'a> Schema<'a> {
      fn execute(&self) -> Vec<&str> {
          self.executes.iter().map(|cmd| cmd()).collect()
      }

  fn add_field() -> &'static str {
      "add field"
  }
  ```

#### 抉擇
如果command較小且可能被定義為function或closure，那就適合使用function pointer，因為不會有動態分配。但若是command是整個struct，有一些function和variable，那使用trait會比較合適。
靜態分配提供較好的效能，但動態分配對於我們程式的結構則較有彈性。

### 3.1.2. Interpreter
如果一個問題經常出現而且需要花費又臭又長的重複步驟才能處理，這個問題可能可以用簡單的語言來表達，並且用一個直譯器來處理。
基本上需要定義：
  * Domain-specific language
  * Grammar 
  * 直譯器

### 3.1.3. Newtype
當單純type aliases不夠用時，使用`Newtype`模式提供<b>type safety</b>和<b>encapsulation</b>。 方法是建立一個tuple struct將原本的type包裝起來，就可以將新的tuple struct加上新功能。

```rust
// Wrapped Type
struct Foo {
    //..
}
// NewType, Wrapper Type
pub struct Bar(Foo);
impl Bar {
  ...
}
```

#### 優點
Newtype是零成本抽象，而且可以將Wrapped type封裝起來，因為存取預設是private。

#### 缺點
這種Rust沒辦法讓Wrapper Type與Wrapped type直接連通，必須自行新增'pass through'函式來暴露你想使用的wrapped type的函式，trait也必須自行實作。

### 3.1.4. RAII Guards
"Resource Acquisition is Initialisation"，在C++很知名的詞也是名字取得很糟的詞。簡單說就是資源在建構時Initialize，然後在解構時finalize。Rust透過RAII的物件作為一個資源的保護，讓解構時可自動執行特定程序。

```rust
struct Foo {}
struct Mutex<T> {
  // ...
}
struct MutexGuard<'a, T: 'a> {
  data: &'a T,
  // ...
}
impl<T> Mutex<T> {
  fn lock(&self) -> MutexGuard<T> {
    // ...
  }
}
impl<'a, T> Drop for MutexGuard<'a, T> {
    fn drop(&mut self) {
      // Unlock
    }
}
fn baz(x: Mutex<Foo>) {
  let xx = x.lock(); // MutexGuard
  xx.foo(); // method on Foo
  // xx在這結束後就會自動走進drop，並且進行lock釋放
}
```

### 3.1.5. Strategy (aka Policy)
關注點分離的一個技巧。基本概念是要用演算法來解決某個特定問題時，我們只抽象定義演算法的架構，然後將實作分到不同部份進行。這樣client就不必依賴某個特定的演算法實作，也就是所謂的"Dependency Inversion"。

```rust
// 抽象演算法
trait Formatter {
    fn format(&self, data: &Data, buf: &mut String);
}
struct Report;
impl Report {
  fn generate<T: Formatter>(g: T, s: &mut String) {
    // ...
    g.format(&data, s);
  }
}
```

```rust
// 各種實作
struct Text;
impl Formatter for Text {
  // ...
}
struct Json;
impl Formatter for Json {
  // ...
}
```

#### 優點
提供關注點分離。`Report`不須了解`Text`和`Json`的實作也能進行。

#### 缺點
每個strategy都必須實作成至少一個module，所以module數量就會隨之增加。User就必須知道多個strategy間的差異。

### 3.1.6. Visitor
Visitor用來封裝運行在一群異質物件上的演算法。

```rust
mod ast {
    pub enum Stmt {
        Expr(Expr),
        Let(Name, Expr),
    }
    pub struct Name {
        value: String,
    }
    pub enum Expr {
        IntLit(i64),
        Add(Box<Expr>, Box<Expr>),
        Sub(Box<Expr>, Box<Expr>),
    }
}
mod visit {
    use ast::*;
    pub trait Visitor<T> {
        fn visit_name(&mut self, n: &Name) -> T;
        fn visit_stmt(&mut self, s: &Stmt) -> T;
        fn visit_expr(&mut self, e: &Expr) -> T;
    }
}
struct Interpreter;
impl Visitor<i64> for Interpreter {
    fn visit_name(&mut self, n: &Name) -> i64 { panic!() }
    fn visit_stmt(&mut self, s: &Stmt) -> i64 {
        match *s {
            Stmt::Expr(ref e) => self.visit_expr(e),
            Stmt::Let(..) => unimplemented!(),
        }
    }
    fn visit_expr(&mut self, e: &Expr) -> i64 {
        match *e {
            Expr::IntLit(n) => n,
            Expr::Add(ref lhs, ref rhs) => self.visit_expr(lhs) + self.visit_expr(rhs),
            Expr::Sub(ref lhs, ref rhs) => self.visit_expr(lhs) - self.visit_expr(rhs),
        }
    }
}
```

## 3.2. Creational
處理物件建立的機制。

### 3.2.1. Builder
透過呼叫Builder helper建立物件。一般用在需要很多建構子或是建構有副作用的時候。因為Rust缺乏overloading，所以這種模式會很常用到。

```rust
pub struct Foo {
  bar: String,
}
impl Foo {
  pub fn builder() -> FooBuilder {
    FooBuilder::default()
  }
}
pub struct FooBuilder {
  bar: String,
}
impl FooBuilder {
  // ...
  pub fn name(mut self, bar: String) -> FooBuilder {
    self.bar = bar;
    self
  }
  pub fn builder(self) -> Foo {
    Foo {bar: self.bar }
  }
}

fn bilder_test() {
  let foo = FooBuilder::new().name(String::from("Y")).build();
}
```

#### 優點
將各個參數建構分開，避免生出多種建構子。

#### 缺點
比起直接建立struct或是簡單的建構函式來得複雜。

### 3.2.2. Fold
類似於Visotor模式，也是跑一個演算法在一群物件上，但是Fold會產生出另一批新的物件。

```rust
// The data we will fold, a simple AST.
mod ast {
    pub enum Stmt {
        Expr(Box<Expr>),
        Let(Box<Name>, Box<Expr>),
    }
    pub struct Name {
        value: String,
    }
    pub enum Expr {
        IntLit(i64),
        Add(Box<Expr>, Box<Expr>),
        Sub(Box<Expr>, Box<Expr>),
    }
}
mod fold {
    use ast::*;
    pub trait Folder {
        fn fold_name(&mut self, n: Box<Name>) -> Box<Name> { n }
        fn fold_stmt(&mut self, s: Box<Stmt>) -> Box<Stmt> {
            match *s {
                Stmt::Expr(e) => Box::new(Stmt::Expr(self.fold_expr(e))),
                Stmt::Let(n, e) => Box::new(Stmt::Let(self.fold_name(n), self.fold_expr(e))),
            }
        }
        fn fold_expr(&mut self, e: Box<Expr>) -> Box<Expr> { ... }
    }
}
```