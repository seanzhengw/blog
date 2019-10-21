---
layout: post
title:  "Pistache - C++ Rest Framework, Environment set up"
date:   2019-04-07 18:00:00 +0800
categories: cpp
tags: [cpp, rest]
---

### 作業系統

Pistache 目前只支援 Linux，因為它使用了 Linux 的系統呼叫 `epoll(7)`

MacOS/BSD  不支援，但有機會以 `kqueue(2)` 替代 `epoll(7)`，詳見 [issues: Pistache work on mac?](https://github.com/oktal/pistache/issues/1)

Windows 不支援，除非 Windows 提供 Reactor 模式的系統呼叫，詳見 [issues: Windows support](https://github.com/oktal/pistache/issues/6)

## 編譯函式庫

首先依照 [Pistache 的 Github](https://github.com/oktal/pistache) 下載與編譯。

### 問題一

首先會遇到的問題應該是執行了編譯步驟的第二步 (在編寫這篇文的時候其步驟還沒修改)

    git submodule update --init

這時候會出現錯誤訊息

    fatal: not a git repository (or any of the parent directories): .git

因為這行命令應該要在 pistache 路徑下執行

### 問題二

如果在使用 cmake 時發生問題，代表 cmake 版本過舊或是沒有安裝 cmake 

解決方法就是安裝 cmake

### 問題三

如果在使用 make 時發生問題，代表沒有安裝 C++ 編譯器

解決方法就是安裝 g++ 或是 clang/clang++

### 更新動態函式庫快取

使用指令 `ldconfig` 更新動態函式庫快取

    sudo ldconfig

## Hello, World!

一樣直接使用 [Pistache 的 Github](https://github.com/oktal/pistache) 最下面的範例建立一個 `main.cpp`。

### 編譯

    g++ main.cpp -o helloworld -lpistache

### 執行

    ./helloworld

### 測試

以瀏覽器開啟 [http://127.0.0.1:9080](http://127.0.0.1:9080)

有出現 Hello, World! 就代表環境都設定好了。

## 使用 CMake 編譯

若整個專案變得更大更複雜，使用 CMake 編譯更方便好管理，這裡不介紹 CMake 的使用，只提供基本的 `CMakeLists.txt` 檔案內容

    cmake_minimum_required(VERSION 3.14)
    project(helloworld)
    add_executable(helloworld main.cpp)
    target_link_libraries(helloworld pistache)
    
### 問題

如果用上面的 CMake 在編譯後執行失敗並顯示這個錯誤訊息

    ./helloworld: error while loading shared libraries: libpistache.so.0: cannot open shared object file: No such file or directory

代表沒有更新動態函式庫快取，解決方法就是使用指令 `ldconfig` 更新動態函式庫快取

    sudo ldconfig

### 靜態連結

將 `CMakeLists.txt` 改成這樣

    cmake_minimum_required(VERSION 3.14)
    project(helloworld)
    add_executable(helloworld main.cpp)
    find_package(Threads REQUIRED)
    add_library(pistache STATIC IMPORTED)
    set_property(TARGET pistache PROPERTY IMPORTED_LOCATION /usr/local/lib/libpistache.a)
    target_link_libraries(helloworld pistache Threads::Threads)

就可以以靜態連結的方式編譯
