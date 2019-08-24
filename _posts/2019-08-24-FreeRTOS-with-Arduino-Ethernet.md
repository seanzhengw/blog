---
layout: post
title:  "STM32 FreeRTOS with Arduino Ethernet"
date:   2019-08-24 18:00:00 +0800
categories: stm32
tags: [stm32, freertos, ethernet]
---

這篇是記錄一些我在 STM32 上同時使用 FreeRTOS 和 Arduino Ethernet 時遇到的問題，這些問題大多是因為 Arduino Ethernet 是使用 SPI 介面與 W5100 通訊，，或是阻擋中斷造成的。

## 問題一：多 Task 使用 SPI 介面

這個問題是當程式中有多個 Task 都會使用到 Ethernet 功能(或共用 SPI 介面)時，在 Ethernet 內部正在使用 SPI 與 W5100 通訊時被 FreeRTOS 中斷並切到其他 Task，而這個 Task 也使用了 SPI 介面，那就會造成上一個 Task 的 Ethernet 方法通訊失敗。

### 解決方法一：禁止 Task 搶佔

將 `FreeRTOSConfig.h` 內的 `#define configUSE_PREEMPTION 1` 改為 `#define configUSE_PREEMPTION 0`，這樣在 Task 自己呼叫暫停的函式前就不會進行切換，用這個解決方法要注意的是程式有沒有要讓某個 Task 高權限的運作，若禁止 Task 搶佔，FreeRTOS 就不會因為目前 Task 執行過久而進行切換，只會在目前 Task 自己暫停時優先切換至高權限的 Task。

### 解決方法二：臨界區段

在呼叫 Ethernet library 的方法之前先呼叫 `taskENTER_CRITICAL()` 避免 FreeRTOS 切換 Task，接著在 Ethernet 呼叫結束後呼叫 `taskEXIT_CRITICAL()` 恢復 Task 切換，用這個方法要注意的點是 `taskENTER_CRITICAL()` 除了禁止 Task 切換之外還會禁止低權限的系統中斷，要注意 `FreeRTOSConfig.h` 內的 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 與 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` 是否符合所需要的中斷情境。

### 解決方法三：調度鎖

這個方法與方法二類似，但是只禁止 FreeRTOS 切換 Task，那就是使用 `vTaskSuspendAll();` 與 `xTaskResumeAll();`，一樣是在呼叫 Ethernet library 的方法之前先呼叫 `vTaskSuspendAll();`，接著在 Ethernet 呼叫結束後呼叫 `xTaskResumeAll();`。

## 問題二：EthernetClient::stop()

在問題一時提到的解決方法二，在 `EthernetClient::stop()` 反而會使 Ethernet 底層卡在一個等待的循環，我目前也還沒研究出問題出在哪裡，但是有找到暫時繞過這個問題的作法：

    // 寫在某處的 EthernetClient
    EthernetClient mClient

    // 如果 mClient 已斷線，呼叫 EthernetClient::stop() 關閉其 socket
    void ClientStopIfDiscon() {
        // 進入臨界區段來呼叫 EthernetClient::connected()
        taskENTER_CRITICAL();
        if (!mClient.connected())
        {
            // 將已經斷線的 client 暫存
            EthernetClient tempClient = mClient;
            // 將 mClient 設為不可用連線 (socket number = MAX_SOCK_NUM)
            mClient = EthernetClient();
            // 先解除 Task Critical
            taskEXIT_CRITICAL();
            tempClient.stop();
            // 再次進入 Task Critical 配合底下的 taskEXIT_CRITICAL()
            taskENTER_CRITICAL();
        }
        taskEXIT_CRITICAL();
    }

這樣就可以避免 `EthernetClient::stop()` 卡在等待循環中，先暫存原本的 mClient 為 tempClient 是為了避免 `taskEXIT_CRITICAL();` 後剛好切換 Task 造成 mClient 被其他 Task 拿去使用，`mClient = EthernetClient();` 這行可以使 mClient 的 socket number 變成不可用，這樣就算其他 Task 使用 mClient 也無法進行操作，**但這樣的做法依賴於 EthernetClient 在執行方法時都會先檢查 socket number，若他未來有修改，這個方法就不一定有效了**，所以還是建議使用**調度鎖**的方法。