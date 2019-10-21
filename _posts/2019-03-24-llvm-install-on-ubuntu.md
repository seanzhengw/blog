---
layout: post
title:  "LLVM 安裝"
date:   2019-03-24 18:00:00 +0800
categories: cpp
tags: [cpp, llvm]
---

### LLVM

下載及解壓縮後用 cmake 配置並編譯安裝

    mkdir build && cd build
    cmake .. -DCMAKE_BUILD_TYPE=Release
    sudo make install

如果覺得編譯有點久可以嘗試用 `sudo make install -j4` 或 `-j6` 看主機規格

### Clang

安裝步驟和 **llvm** 的步驟一樣

另外要設定環境變數，使 CMake 配置的 Makefiles 使用 clang/clang++

    echo 'export CC=clang' >> ~/.bashrc
    echo 'export CXX=clang++' >> ~/.bashrc

### LLD

安裝步驟和 **llvm** 的步驟一樣

最後軟鏈結 ld.lld 為 ld，可以使 clang 固定使用 lld

    sudo ln -s /usr/local/bin/ld.lld /usr/local/bin/ld

### 自舉編譯

現在可以用 llvm + clang + lld 這個組合重新編譯一次自己，當然不這麼做也沒關係

### LLDB

在編譯之前要先安裝 LLDB 依賴的函式庫

    sudo apt-get install build-essential subversion swig python2.7-dev libedit-dev libncurses5-dev

接著就可以開始安裝，安裝步驟和 **llvm** 的步驟一樣，但這次在 cmake 配置時會顯示有關於 llvm-lit 的警告，但是還是可以正常編譯安裝
