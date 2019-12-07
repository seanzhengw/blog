---
layout: post
title:  "CentOS 8 安裝 Docker"
date:   2019-11-30 18:00:00 +0800
categories: docker
tags: [docker]
---

# 簡述

在 CentOS 8 安裝 Docker 的紀錄。

最近決定在另外一台 CentOS 8 的主機上用 Docker 裝 GitLab Runner，但 Docker 安裝時有點問題。

另外這裡改用 `dnf` ，所以與 Docker 官網教學有一點點指令上的差異 (官網上還是用 `yum`)

### 官網教學

[https://docs.docker.com/engine/install/centos/](https://docs.docker.com/engine/install/centos/)

#### 改用 dnf

第一個步驟是新增儲存庫，使用 `yum` 的時候就像教學上這樣

    sudo yum install -y yum-utils
    sudo yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo

如果改用 `dnf` ，那並不需要先安裝 `yum-utils`，並且將 `yum-config-manager` 的地方改成 `dnf config-manager`，要注意 `yum-config-manager` 是指令，`dnf config-manager` 是指令 `dnf` + 參數 `config-manager`

    sudo dnf config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo

接下來的步驟中直接將 `yum` 替換為 `dnf` 即可。

#### 問題

目前依照 Docker 官網的教學去安裝會遇到問題

    [root@localhost ~]# dnf install docker-ce docker-ce-cli containerd.io

    ...

    Error:
     Problem: package docker-ce-3:19.03.11-3.el7.x86_64 requires containerd.io >= 1.2.2-3, but none of the providers can be installed
      - cannot install the best candidate for the job
      - ...
      - ... (這裡會列出數個 containerd.io-x.x.x 版本)
      - ...
    (try to add '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)

這應該只是官方給的倉庫相依性設定上的問題，畢竟確實有 `containerd.io-1.2.2-3.3.el7.x86_64` 以及更高的版本可以安裝

要注意不能使用 `--skip-broken`，這會變成不安裝 `containerd.io`，所以應該要改成這樣安裝

    [root@localhost ~]# dnf install docker-ce docker-ce-cli containerd.io --nobest

`--nobest` 就不會依照倉庫設定的相依性去選擇版本

#### 檢查

發生這個問題時 `docker --version` 是可以正常輸出版本訊息的，

但是 docker 卻無法正常使用，

並且 `/usr/lib/systemd/system/docker.service` 與 `/usr/lib/systemd/system/docker.socket` 並沒有被建立。

所以用 `systemctl start docker` 去判斷正不正常就好。

### 防火牆設定

為了讓 docker 容器能正常存取外部網路，需要將 docker0 新增到防火牆的 trusted 區

    firewall-cmd --permanent --zone=trusted --change-interface=docker0
    firewall-cmd --reload
    systemctl restart docker

直接信任 docker0 是比較簡便的作法，若是對於容器內的外部網路存取權有比較多的安全性要求，也可以用 firewall direct rules 設定。
