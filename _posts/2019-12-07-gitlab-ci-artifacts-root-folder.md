---
layout: post
title:  "GitLab CI 的 artifacts 根目錄"
date:   2019-12-07 18:00:00 +0800
categories: ci
tags: [gitlab, ci]
---

# 問題

GitLab CI 的 artifacts 根目錄目前還無法指定，可見 [gitlab-runner #1057](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/1057)，這是一個開放討論近四年卻未有結論的問題。

GitLab CI 會以 CI 專案目錄當作 artifacts 的根目錄，舉例來說，我的 repo 在經過自動處理之後的狀態如下

    /my-repo/a.txt
    /my-repo/output/b.txt

其中 `a.txt` 是專案源檔， `output/b.txt` 是專案的輸出檔

而我的 `gitlab-ci.yml` 關於內設定如下

    texts:
      stage: build
      script:
        (...)
      artifacts:
        name: "texts"
        paths:
        - output/b.txt
        expire_in: 1 month

這樣產出的工件 texts.zip 結構會是

    texts.zip/output/b.txt

但是我實際上想要的狀態是

    texts.zip/b.txt

## 暫時解法

目前 GitLab CI 還沒有提供這樣的功能，想要達成上面這樣的輸出方式，目前只能自己在 script 部分去移動輸出的檔案，例如這樣

    texts:
      stage: build
      script:
        (...)
        - mv output/b.txt b.txt
      artifacts:
        name: "texts"
        paths:
        - b.txt
        expire_in: 1 month

這樣就可以達成輸出的結構是 `texts.zip/b.txt`

### 問題

這樣直接移動檔案會遇到檔案名稱衝突，例如上面的例子輸出檔案如果也命名為 `a.txt` 時， `mv output/a.txt a.txt` 將會失敗，在 [#1057](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/1057) 中也有一大部分在討論，討論如果可以在設定檔中指定多個不同的根目錄，如何完善的處裡檔案名稱衝突的問題，這和直接使用 `mv` 指令的問題幾乎是相同的。
