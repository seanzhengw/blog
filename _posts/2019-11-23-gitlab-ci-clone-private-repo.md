---
layout: post
title:  "GitLab CI 如何 clone private repository"
date:   2019-11-23 18:00:00 +0800
categories: ci
tags: [gitlab, ci]
---

# 簡述

使用 GitLab CI 時需要 clone private repository 該如何設定。

### 方法1：使用 GitLab 建議的方式

參考 [Using Git submodules with GitLab CI](https://docs.gitlab.com/ce/ci/git_submodules.html)

簡單來說就是使用 git submodule 功能，並將 submodule 的 url 改為相對路徑，這樣 gitlab-runner 就可以將 submodule clone 下來

這個方法中的 private repo 與要用到 gitlab-ci 的 repo 必須位於同一個 gitlab 網頁伺服器上(例如自建的 gitlab-ce/ee，或是都託管在 gitlab.com 上)

##### 範例

1. 自建的 gitlab-ce 位於 192.168.0.123
2. 沒有設定 HTTPS
3. 我的 repo: http://192.168.0.123/myaccount/myproject.git
4. 有一個 private repo: http://192.168.0.123/mygroup/mymodule.git
5. 加入 submodule

        git submodule add http://192.168.0.123/mygroup/mymodule.git mymodule

6. 將 .gitmodule 內的 url 改為相對路徑

        [submodule "mymodule"]
          path = mymodule
          url = ../../mygroup/mymodule.git

7. 在 gitlab-ci.yml 設定要取用 submodule

        variables:
          GIT_SUBMODULE_STRATEGY: recursive

   或是自己手動取用 submodule

        before_script:
          - git submodule sync && git submodule update --init

##### 優點

* 不需要更動伺服器端的設定
* 非系統管理員也可以使用
* 不需要自建 gitlab-runner

##### 缺點

* 強迫使用 git submodule
* 用相對路徑會使 git submodule 在本地端出錯
* 各式開發用框架的自動化配置若是使用 git clone 將無法正常運作

### 方法2: 自建有讀取權的 gitlab-runner

直接設定一組 gitlab 帳戶在 gitlab-runner 中使用

這個方法不論是直接使用 HTTP BASIC 或是設定 SSH Keys 都可以

##### 優點

* 不需要為了取得 private repo 而去修改 gitlab-ci.yml
* 不強迫使用 git submodule
* 可以讓 gitlab-ci 自動取用 git submodule (GIT_SUBMODULE_STRATEGY: recursive)
* 可以寫 script 取用 git submodule
* 各式開發用框架的自動化配置若是使用 git clone 也能正常運作

##### 缺點

* 限定用於自建的 gitlab-runner (除非本身是 gitlab-ce 的主機管理員，可以建 Shared Runner)
* 需要一台主機，以及一個 gitlab 伺服器可見的 IP 位址才能自建 gitlab-runner
* 由於這個 gitlab-runner 會擁有帳戶權限，需要詳細設定帳戶權限，並控管使用者，避免被惡意的 ci script 攻擊(例如嘗試 git push --force 去覆蓋 server 上的 repo，或是隨意建 tag 上傳)。

### 方法3：使用 Access Token

建立一個可以用於讀取 private repo 的帳戶 (用原本的帳戶也可以)

並新增具有 read_repository 權限的 Access Token 來讀取 private repo

##### 範例

1. 自建的 gitlab-ce 位於 192.168.0.123
2. 沒有設定 HTTPS
3. 建立帳號 myciaccount
4. 建立 Access Token: XXXXXXXXXXXXXXXXXXXX

    > 要注意帳戶有啟用 2FA 則必須使用 Access Token 當作 HTTP BASIC 的密碼，若沒啟用 2FA 則 Access Token 與密碼都可以使用。

5. 在 gitlab-ci.yml 設定

        before_script: 
          - git config --global url."http://${CI_ACCOUNT}:${CI_ACCESS_TOKEN}@192.168.0.123/".insteadOf "http://192.168.0.123/"

6. 在專案的 Settings -> CI/CD 裡面的 Variables 加入兩個變數

    | Type | Key | Value | State | Masked |
    | -- | -- | -- | -- | -- |
    | Variable | CI_ACCOUNT | myciaccount | Protected X | Masked ✓ |
    | Variable | CI_ACCESS_TOKEN | XXXXXXXXXXXXXXXXXXXX | Protected X | Masked ✓ |

    > 其中 Protected 的部分就看專案需求，Masked 務必啟用避免 public pipeline 顯示 token。

##### 優點

* 不論是 Shared、Group 或 Specific Runner 都可以使用這個方法
* 不需要自建 gitlab-runner
* 由 Access Token 限定為只能讀取
* 不強迫使用 git submodule
* 可以寫 script 取用 git submodule
* 各式開發用框架的自動化配置若是使用 git clone 也能正常運作

##### 缺點

* 需要稍微修改 gitlab-ci.yml 以使用 HTTP BASIC 認證去取得 private repo
* 不能讓 gitlab-ci 自動取用 git submodule (GIT_SUBMODULE_STRATEGY: recursive)，因為 gitlab-ci 的 Git submodules setup 步驟是在 job 之前，這時候 git url 替換的指令尚未設定
* 每個專案都需要設定一次 CI/CD Variables

---

建議用方法3，不需要自建 runner，且不論在私有或公共 gitlab-ce/ee 或 gitlab.com 上都可以使用，也不強迫要使用 submodule，是三種方式裡面限制最少的。

---

其實 gitlab 有內建一個變數 CI_JOB_TOKEN ，但這個 Token 看起來只能讀取目前的 repo，有跟沒有一樣。
