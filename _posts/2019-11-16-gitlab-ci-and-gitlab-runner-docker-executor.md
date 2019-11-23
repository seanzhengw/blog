---
layout: post
title:  "GitLab CI 與 GitLab Runner、Docker Executor"
date:   2019-11-16 18:00:00 +0800
categories: ci
tags: [gitlab, ci]
---

# 簡述

要使用 GitLab CI 需要一個 GitLab Runner，GitLab Runner 會接收 GitLab Server 上發送的工作請求並回覆執行結果。

本篇是關於如何建立一個 GitLab Runner，並將其設定為啟動一個 Docker 容器作為工作環境來執行專案的 GitLab CI 設定。

### 跨平台

Runner 是獨立於 Server 運作的(是另一個 Process)，Runner 與 Server 透過網路連接，也就是說可以將 Runner 安裝在另一個主機上，若將 Runner 安裝在 windows 主機上就可以利用 windows 的環境來建置專案，且這也只是 Runner 跨平台的其中一種方式。

### 共享與專用 Runner

可以建立數個 Runner 給不同的專案共享或專用，Server 的系統管理員可以建立共享的 Runner 給整個 Server 上的專案使用，這樣大多數使用者不必再另外設定 Runner，而需要特殊 Runner 的使用者可以為自己的專案建立專用的 Runner。

另外，一般來說不建議 Server 的系統管理員將 Runner 安裝在 Server 的主機上，雖然將共享 Runner 與 Server 安裝於同一台主機聽起來很合理，但 Runner 執行的工作是按照專案內的要求，需要的記憶體空間與處理器效能是不可掌控的，例如在建置專案時編譯的多核心選項開到最高時，Runner 可能會消耗過多的處理器效能，Runner 也可以設定為根據目前的工作量自動縮放，可以說 Runner 主機所需要的硬體規格會比 Server 高出不少，為了維持 Server 的穩定最好是將 Runner 安裝於其他主機上，但如果只是供個人使用的話影響並不大。

### 執行器

執行器是設定 Runner 將以何種方式執行工作，Runner 有許多執行器如：

* Shell Executor: 直接在 Runner 的主機上執行命令
* SSH Executor: 透過 SSH 連線到與端主機上執行命令
* Docker Executor: 從指定映像檔啟動一個容器，並執行命令
* Virtual Machine Executor: 複製(clone)一個現有的虛擬主機，啟動並連線至虛擬主機內執行命令，這可以簡單的達成單實體主機跨多平台建置。
* Kubernetes Executor: 透過 Kubernetes cluster API 自動為工作建立 Pod 並建立 build container 執行命令。

這裡主要是紀錄 Docker Executor，其他的可以在 [Executors](https://docs.gitlab.com/runner/executors/README.html) 中查看。

Docker Executor 能提供乾淨的建置環境與比較簡單快速的配置，可以將建置所需要的環境預先放在 Docker image 中，Docker Executor 將以指定的 image 啟動 container 並執行命令，如此也不會將編譯環境弄亂。

# 安裝與設定

### 安裝

依照 [Install](https://docs.gitlab.com/runner/install/) 中各作業系統的安裝法。

其實 GitLab Runner 就是一個軟體，安裝完是使用指令 `gitlab-runner` 啟動與停止。

### 使用 Docker 安裝 GitLab Runner

GitLab Runner 也可以作為 Docker 服務執行 (這和上面說的 Docker Executor 是兩回事)，只要用 [gitlab/gitlab-runner](https://hub.docker.com/r/gitlab/gitlab-runner) 這個 image 就可以了。

    docker run -d \
    --name gitlab-runner-docker \
    --restart always \
    -v /srv/gitlab-runner/config:/etc/gitlab-runner \
    -v /var/run/docker.sock:/var/run/docker.sock \
    gitlab/gitlab-runner:latest

* `-d`: 啟動為背景服務
* `--name`: 容器名稱
* `--restart always`: 自動重啟
* `-v /srv/gitlab-runner/config:/etc/gitlab-runner`:
  gitlab-runner 的設定檔 `config.toml` 會在容器內的 `/etc/gitlab-runner` 路徑下，將他聯結到主機空間 `/srv/gitlab-runner/config`
* `-v /var/run/docker.sock:/var/run/docker.sock`:
  因為要讓 `gitlab-runner-docker` 這個容器可以管理 docker，才能夠讓 Docker Executor 啟動其他容器來執行工作，所以要將 `/var/run/docker.sock` 聯結
  
### 向 Server 註冊

安裝好 gitlab-runner 後要向 GitLab 註冊這個 Runner

1. 執行命令，接下來會以問答式的方式填入註冊訊息
   這裡是 GitLab Docker Runner 的執行方式

        docker exec -it gitlab-runner-docker gitlab-runner register

   如果是直接安裝在主機上的的 `gitlab-runner` 要使用命令

        sudo gitlab-runner register

2. GitLab 主機 URL，這裡可以輸入自己架設的 GitLab 主機網址

        Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com )
        http://127.0.0.1

3. 輸入註冊 token，
   共享 Runner 的 token 在 GitLab > Admin Area > Overview > Runners
   專用 Runner 的 token 在 Project Settings > CI/ CD > Runners
   
        Please enter the gitlab-ci token for this runner
        xxxxxxxxxxxxxxxxxxxx

4. 輸入 Runner 的描述，類似 Runner 的名稱，描述可以在 GitLab 中修改
   [...] 是 Runner 主機名稱，若不輸入值就會直接以主機名稱作為描述

        Please enter the gitlab-ci description for this runner
        [...]: my-runner

5. 輸入 Runner 的 tag，專案的工作可以設定只使用具有某個 tag 的 Runner 執行
   這可以很方便的用來建立許多不同環境的 Runner 讓不同專案使用。

        Please enter the gitlab-ci tags for this runner (comma separated):
        tag1,mytag

6. 輸入 Runner 的 Executor，這一步就是在設定 Runner 的執行器
   這裡選擇 Docker Executor 所以輸入 docker

        Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:
        docker

7. 選擇 Docker Executor 後會詢問，如果專案沒有在設定檔中指定映像檔時，要使用哪一個映像檔當作預設環境
        
        Please enter the Docker image (eg. ruby:2.1):
        buildpack-deps:buster

    * 要注意的是這是建置用的環境，若沒有能力設定完整的建置環境就不要使用精簡版的映像檔，如 alpine，雖然 alpine 很省空間，但程式語言編譯的環境也都精簡了，只會造成使用上的麻煩，而且將建置環境安裝進去實際上不會節省太多空間。
    * 這裡推薦使用 `buildpack-deps` ，這個映像檔已經先安裝好許多工具與函式庫，`buildpack-deps` 是基於 Debian 的映像檔，缺少的工具與環境也都可以用 `apt-get` 安裝。
    * 如果是為了特定的發行版建置，也可以直接使用發行版的映像檔。

### 設定 Docker Executor 使用本機映像檔

若想要使用自己在 Runner 主機上建置的 Docker 映像檔，需要設定 Docker Executor 優先使用本地端的映像檔，否則會在 GitLab CI 上看到這樣的錯誤訊息: (在專案CI檔中指定使用 myimage)
<b><span style="color:red">
ERROR: Job failed: Error response from daemon: pull access denied for myimage, repository does not exist or may require 'docker login': denied: requested access to the resource is denied</span></b>

這時候要修改設定檔 `/etc/gitlab-runner/config.toml`，在 `[runners.docker]` 中加入

    pull_policy = "if-not-present"

之後 `config.toml` 看起來應該是這樣:

    name = "my-runner"
    url = "http://127.0.0.1/"
    token = "xxxxxxxxxxxxxxxxxxxx"
    executor = "docker"
    [runners.custom_build_dir]
    [runners.docker]
        tls_verify = false
        image = "buildpack-deps:buster"
        privileged = false
        disable_entrypoint_overwrite = false
        oom_kill_disable = false
        disable_cache = false
        volumes = ["/cache"]
        shm_size = 0
        pull_policy = "if-not-present"
    [runners.cache]
        [runners.cache.s3]
        [runners.cache.gcs]

這樣 Docker Executor 就會只在本地端沒有映像檔的時候才嘗試 pull。

---

至於 GitLab CI 要怎麼玩不是本篇重點。

