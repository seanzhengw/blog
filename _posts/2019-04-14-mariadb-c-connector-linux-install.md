---
layout: post
title:  "MariaDB C&C++ Connector install for linux"
date:   2019-04-14 18:00:00 +0800
categories: cpp
tags: [cpp, mariadb]
---

從 [MariaDB](https://mariadb.com/downloads/#connectors) 官網下載的 C/C++ Connector 是編譯好的二進制版本，只要將檔案各自放到正確地資料夾就可以使用。

## 全域安裝

就像一般從原始碼編譯並使用 `make install` 做的事情一樣，將下載的檔案解壓縮後會有三個資料夾，分別是 `bin`、`include`、`lib`，將這三個資料夾複製到 `/usr/local` 即可將 Connector 安裝於系統內。

其中 `bin/mariadb_config` 是用於取得各項編譯器參數，不放到 `/usr/local/bin` 也不影響使用

## 只用於特定專案

如果不想將 Connector 安裝於系統內，只是想在某個專案中使用的話，就在編譯與連結時加上路徑即可。

需要連結的參數可以執行解壓縮資料夾內的 `./bin/mariadb_config` 取得

專案結構

    myproj/
    ├── include/
    ├── lib/
    └── src/
        └── main.cpp

測試用的程式 `main.cpp`

    #include <iostream>
    #include <mariadb/mysql.h>

    int main()
    {
        MYSQL *conn = mysql_init(NULL);

        conn = mysql_real_connect(conn, "127.0.0.1", "root", "password", "test", 3306, NULL, 0);
        if (conn == NULL)
        {
            std::cout << mysql_error(conn) << std::endl;
            exit(1);
        }
        else

        std::cout << "Connect to MariaDB 127.0.0.1@root" << std::endl;
        std::cout << "SQL Query: CREATE DATABASE IF NOT EXISTS testdb;" << std::endl;
        if (mysql_query(conn, "CREATE DATABASE IF NOT EXISTS testdb;"))
        {
            std::cout << mysql_error(conn) << std::endl;
            mysql_close(conn);
            exit(1);
        }

        std::cout << "Press any key..." << std::endl;
        std::cin.ignore();
        mysql_close(conn);
    }

這個程式將會以帳號 `root` 與密碼 `password` 登入 `127.0.0.1:3306@test`，若登入成功就會以 SQL `CREATE DATABASE IF NOT EXISTS testdb;` 建一個名為 `testdb` 的資料庫，最後按任意鍵退出。

在環境變數 `LD_LIBRARY_PATH` 加上專案的 `lib` 資料夾完整路徑

    export LD_LIBRARY_PATH=/home/user/myproj/lib/mariadb:$LD_LIBRARY_PATH

如果沒有加 `lib` 路徑的話在執行編譯的執行檔時會顯示錯誤

    ./a.out: error while loading shared libraries: libmariadb.so.3: cannot open shared object file: No such file or directory

## 編譯

#### g++ 

    g++ src/main.cpp -Iinclude/ -Llib/mariadb -lmariadb

#### clang++
    
    clang++ src/main.cpp -Iinclude/ -Llib/mariadb -lmariadb
    