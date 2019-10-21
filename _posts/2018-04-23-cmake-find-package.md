---
layout: post
title:  "CMake find_package 使用"
date:   2018-04-23 18:00:00 +0800
categories: cmake
tags: [cpp, cmake]
---

## 範例

這裡以 Boost 函式庫為例

在使用 CMake 來管理專案時，可以使用 `find_package()` 來自動尋找函式庫與路徑

在 CMake 執行到 `find_package(Boost 1.36.0)` 之後，若有找到，會設定以下變數

    Boost_FOUND            - True 如果有找到標頭檔與需要的函式庫檔案
    Boost_INCLUDE_DIRS     - Boost 的標頭檔路徑
    Boost_LIBRARY_DIRS     - Boost 的外部函式庫路徑
    Boost_LIBRARIES        - Boost 函式庫
    Boost_<C>_FOUND        - True 如果特定的函式庫 <C> 有找到 (<C> 是大寫名稱)
    Boost_<C>_LIBRARY      - Boost 內的特定函式庫 <C>
    Boost_VERSION          - Boost 寫在 boost/version.hpp 的版本號
    Boost_LIB_VERSION      - Boost 函式庫檔案名稱上的版本號
    Boost_MAJOR_VERSION    - Boost 主版本號 (X.y.z 的 X)
    Boost_MINOR_VERSION    - Boost 次版本號 (x.Y.z 的 Y)
    Boost_SUBMINOR_VERSION - Boost 次版本號 (x.y.Z 的 Z)
    Boost_LIB_DIAGNOSTIC_DEFINITIONS (Windows)
                           - 這只有 Windows 有，這個變數可以傳遞給 add_definitions()
                             ，來設定 Boost 的編譯時期自動鏈結的 Windows 診斷資料

所以 `CMakeLists.txt` 可以這樣寫

    project(foo)
    # 搜尋 Boost 函式庫 1.36.0 版及相關路徑
    find_package(Boost 1.36.0)
    # 這樣寫可以針對 regex 函式庫搜索
    # find_package(Boost 1.36.0 REQUIRED COMPONENTS regex)
    if(Boost_FOUND)
        # 引用 Boost 的標頭檔資料夾
        include_directories(${Boost_INCLUDE_DIRS})
        # 設定連結器搜尋 Boost 函式庫路徑
        link_directories(${Boost_LIBRARY_DIRS})
        # 建立執行檔 foo
        add_executable(foo main.cpp)
        # 鏈結函式庫
        target_link_libraries(foo ${Boost_LIBRARIES})
    endif()

注意 Windows 作業系統下 `find_package` 是依靠系統變數 `<Package>_ROOT` 來搜尋函式庫，以 Boost 來說，要設定環境變數

    Boost_ROOT = C:\cpp\build_boost_1_39_0

實際路徑要看 Boost 在安裝時下的指令 `b2 install --prefix=<PATH>` 而定

## 其他函式庫

上面的範例可以用 `find_package(Boost 1.36.0)` 是因為 CMake 內建了一個名為 `FindBoost` 的模組

要查看 CMake 內建那些模組可以使用指令

    cmake --help-module-list

可以用 grep 過濾出以 Find 關鍵字開頭的模組

    cmake --help-module-list | grep -E ^Find

只要有內建 `Find<Package>` 的模組，就可以在 `CMakeLists.txt` 裡面使用  `find_package(<Package>)` 來自動找函式庫與相關路徑

和 Boost 的範例一樣，如果使用 Windows 作業系統，那麼 `Find<Package>` 模組通常是依靠系統變數 `<Package>_ROOT` 來搜尋函式庫

## 沒有對應的 `Find<Package>` ?

可以自己寫 `Find<Package>`，位置可以放在要執行 `cmake` 指令的路徑，也可以放在 cmake 的 Modules 路徑下供全域使用

內容可以參考其他 `Find<Package>` 模組的寫法，通常都是先在系統安裝路徑搜索，然後再到環境變數 `<Package>_ROOT` 路徑下搜索，最後像 `find_package(Boost 1.36.0)` 後會設定那些變數一樣。
