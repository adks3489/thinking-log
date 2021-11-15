---
title: Rust Design Pattern 讀後整理 - Functional Usage of Rust
date: 2021-11-14 21:02:32
tags: 
  - rust
  - design pattern
---
# 5. Functional Programming
Rust是一個指令式的語言，但也有一些functional programming的範式。
<!-- more -->
## 5.1. Programming paradigms
從指令式語言背景要轉換到functional programming時，最大的障礙就是思維要轉變。指令式語言描述的是<b>如何</b>做某件事，而宣告式則是描述<b>什麼</b>。

### 指令式
```rust
let mut sum = 0;
for i in 1..11 {
    sum += i;
}
println!("{}", sum);
```

### 宣告式
```rust
println!("{}", (1..11).fold(0, |a, b| a + b));
```

## 5.2. Generics as Type Classes
Rust的型別系統設計比較偏向functional language而不是指令式語言，因此可以將許多程式問題轉為靜態型別的問題。這是選擇functional language的一大好處，也是Rust能夠提供許多compile time時期保證的原因。

最主要的部份就是泛型，在C++裡，泛型是由compiler建立的meta-programming產生。`vector<int>`和`vector<char>`就是由一份`vector`程式碼產生的兩個不同的複製體，也就是template。

在Rust則是採用functional language的類型約束，每個不同的參數實際上會改變型態，`Vec<isize>`和`Vec<char>`就是不同的型態。

這情況即為<b>monomorphization(單態化)</b>，不同的型別是由多型的程式碼產生。而不同的類型可以擁有不同的`impl`，即實作區塊。

在物件導向語言，class可以從parent繼承行為，但是單態的作法除了能附加行為，還能轉變成不同的行為。

與這作法最接近的就是Javascript和Python的執行期多型，然而Rust的附加方法可以透過靜態的型別檢查，提供更好可用性的同時還保持安全性。

```rust
mod nfs {
    #[derive(Clone)]
    pub(crate) struct AuthInfo(String);
}
mod bootp {
    pub(crate) struct AuthInfo();
}
mod proto_trait {
    pub(crate) trait ProtoKind {
        type AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo;
    }
    pub struct Nfs {
        auth: nfs::AuthInfo,
        mount_point: PathBuf,
    }
    impl Nfs {
        pub(crate) fn mount_point(&self) -> &Path {
            &self.mount_point
        }
    }
    impl ProtoKind for Nfs {
        type AuthInfo = nfs::AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo {
            self.auth.clone()
        }
    }
    pub struct Bootp();
    impl ProtoKind for Bootp {
        type AuthInfo = bootp::AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo {
            bootp::AuthInfo()
        }
    }
}

struct FileDownloadRequest<P: ProtoKind> {
    file_name: PathBuf,
    protocol: P,
}

impl<P: ProtoKind> FileDownloadRequest<P> {
    fn file_path(&self) -> &Path {
        &self.file_name
    }

    fn auth_info(&self) -> P::AuthInfo {
        self.protocol.auth_info()
    }
}

impl FileDownloadRequest<Nfs> {
    fn mount_point(&self) -> &Path {
        self.protocol.mount_point()
    }
}
```
