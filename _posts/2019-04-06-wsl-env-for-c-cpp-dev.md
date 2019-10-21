---
layout: post
title:  "Windows Subsystem for Linux as C/C++ development environment"
date:   2019-04-06 18:00:00 +0800
categories: cpp
tags: [cpp, wsl]
---

在 Windows Subsystem for Linux (WSL) 推出之前，如果平時有同時使用 Linux 與 Windows 的需求通常就只能在裝虛擬機或雙系統之間選擇，或是有足夠資金就乾脆多買一台電腦，現在有了 WSL 就可以直接在 Windows 環境下進行 Linux 上的 CLI 軟體開發，甚至可以透過 DE 與 XRDP 開啟桌面環境來開發桌面軟體，不再需要受虛擬機或是雙系統的束縛了。

# WSL 內的環境設定

與在 Linux 實機設定相同，以下整理我平常用的環境

***

## Development Tools

### CMake

直接在 [CMake Downloads](https://cmake.org/download/) 下載最新預編譯版本並安裝

以 `cmake-3.14.0-Linux-x86_64.sh` 為例

    sudo mkdir /opt/cmake
    sudo bash cmake-3.14.0-Linux-x86_64.sh --prefix=/opt/cmake
    sudo ln -s /opt/cmake/bin/cmake /usr/local/bin/cmake

### g++

    sudo apt install g++

### llvm & clang

下載後用 cmake 配置並編譯與安裝

    cmake .. -DCMAKE_BUILD_TYPE=Release
    make -j8
    sudo make install

上面第二行的 `-j8` 是使用 8 執行緒編譯，依照電腦處理器調整。

### clang

和 **llvm** 一樣

可以設定系統變數，使 CMake 配置的 Makefiles 使用 clang/clang++

在 ~/.bashrc 檔案最下方加入兩個環境變數

    export CC=clang
    export CXX=clang++

### lld

和 **llvm** 一樣

最後軟鏈結 ld.lld 為 ld，可以使 clang 固定使用 lld

    sudo ln -s /usr/local/bin/ld.lld /usr/local/bin/ld

***

## Libaries

### Google Test

下載後用 cmake 配置並編譯

    cmake .. -DBUILD_GMOCK=OFF
    make
    sudo make install

如果要連 Google Mock 一起裝就不需要加 `-DBUILD_GMOCK=OFF`，或是改為 `ON`

### Boost

下載後

    ./bootstrap.sh --with-toolset=clang
    ./b2 toolset=clang -j8
    sudo ./b2 install

上面第二行的 `-j8` 是使用 8 執行緒編譯，依照電腦處理器調整。

然後更新動態連接庫連結資訊

    sudo ldconfig

***

# Windows 端環境

## 環境變數

建議將 Linux Subsystem 的跟目錄與使用者家目錄加入 Windows 的使用者環境變數
，因為是 Linux Subsystem 是每個使用者各自獨立的，設定為系統環境變數並沒有意義

    WSL_ROOT = C:\Users\<UserName>\AppData\Local\Packages\CanonicalGroupLimited.UbuntuonWindows_79rhkp1fndgsc\LocalState\rootfs
    WSL_HOME = C:\Users\<UserName>\AppData\Local\Packages\CanonicalGroupLimited.UbuntuonWindows_79rhkp1fndgsc\LocalState\rootfs\home\<username>

如果安裝的是其他發布版的 WSL 那路徑的 `CanonicalGroupLimited.UbuntuonWindows_79rhkp1fndgsc` 會不一樣

以 Kali Linux WSL 來為例則會是 `KaliLinux.54290C8133FEE_ey8k8hqnwqnmg`

## Visual Studio Code 

### Terminal

    "terminal.integrated.shell.windows": "C:\\WINDOWS\\System32\\wsl.exe"

## c_cpp_properties.json

配置 WSL 設定，

    {
        "configurations": [
            {
                "name": "WSL",
                "includePath": [
                    "${env:WSL_ROOT}/usr/include",
                    "${env:WSL_ROOT}/usr/local/include",
                    "${workspaceRoot}"
                ],
                "defines": [],
                "intelliSenseMode": "clang-x64",
                "browse": {
                    "path": [
                        "/usr/include",
                        "/usr/local/include",
                        "${workspaceRoot}"
                    ],
                    "limitSymbolsToIncludedHeaders": true,
                    "databaseFilename": ""
                }
            }
        ],
        "version": 4
    }
