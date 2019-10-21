---
layout: post
title:  "CentOS 8 設定 Samba"
date:   2019-10-21 18:00:00 +0800
categories: centos8
tags: [samba]
---

## 安裝

    sudo yum -y install samba

## 設定檔

/etc/samba/smb.conf

    [global]
        workgroup       = SAMBA
        server string   = Samba Server
        security        = user
        passdb backend  = tdbsam

    [homes]
        comment         = User's Home Directories
        browseable      = No
        writable        = Yes
        create mode     = 0664
        directory mode  = 0775

    [public]
        comment         = Public Directories
        path            = <path>
        browseable      = Yes
        writable        = Yes
        create mode     = 0664
        directory mode  = 0775
        write list      = @users


global 是關於 samba server 的設定
* security = user 代表需要帳戶密碼登入，使用者是 linux 使用者，密碼則要另外設定一組 Samba 用的密碼。

homes 是對應 linux 使用者的 home 資料夾，若有一個 linux 使用者 `test123` 登入了 samba，將會看到一個名為 `test123` 的資料夾，這個資料夾就代表 `/home/test123`。
* browseable 代表其他使用者是否能看到這個資料夾
* writable 代表對此資料夾具有存取權的使用者的是否可以寫入檔案，否則唯讀
* create mode 代表使用者建立的檔案權限(linux 檔案權限)
* directory mode 代表使用者建立的目錄權限(linux 目錄權限)

public 會建立一個 public 資料夾(Samba上顯示的資料夾)
* path 要指向主機上的目錄，要注意的是這個目錄需要讓所有使用者都有存取權，將權限設定給 SAMBA 群組可能是個好主意。
* write list 是可以進入這個資料夾的使用者清單，@users 代表所有 SAMBA 使用者。
* 若不需要共享的資料夾，將 public 這部分設定刪除即可。

## 啟用 samba 服務

啟用

    sudo systemctl enable smb nmb

重啟 (修改 smb.conf 後重啟才會生效)

    sudo systemctl restart smb nmb

## 防火牆設定

允許 samba 服務通過防火牆

    sudo firewall-cmd –permanent –zone=public –add-service=samba
    sudo firewall-cmd –reload

## 使用者群組與使用者設定

新增群組 SAMBA

    groupadd SAMBA

將使用者加入 SAMBA 群組

    usermod -a -G SAMBA <username>

設定 Samba 使用者的 Samba 登入密碼 (這個 Samba 使用者必須已經是主機上的 linux 使用者)

    pdbedit -a -u <username>

## SELinux 設定

預設狀況下 SELinux 會封鎖 Samba 使用者存取 home 目錄，只要將 `samba_enable_home_dirs` 設為 on 即可

    sudo setsebool -P samba_enable_home_dirs on
