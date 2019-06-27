---
layout: post
title:  "FreeRTOS Task Size"
date:   2019-05-05 18:00:00 +0800
categories: freertos
tags: [freertos]
---

這篇文是關於 FreeRTOS 的記憶體用量的問題，如果發現你在呼叫 `xTaskCreate()` 之後消耗的記憶體太多，或是明明有 8KB 的記憶體，卻在建立了幾個 stack 128 bytes 的 Task 之後就開始建立失敗，那這篇文可能會讓你找到解法。

## xTaskCreate() stack 

在建立 Task 時要給定一個 stack size，如：

    xTaskCreate(Task1, NULL, 128, NULL, 1, NULL);

這樣表面上是給定 128 的 stack size，但是實際上會分配 `128 * sizeof(StackType_t)` 個 byte 的空間，而這個 `sizeof(StackType_t)` 可能是 2 也可能是 4，以常見的 ARM Cortex-M 系列的微控制器來說這個值會是四，也就是說參數給 128 會分配 128 * 4 = 512 bytes 的空間給這個 Task。

## Task size

另外一個問題則是 Task 本身占用多少記憶體，FreeRTOS 網站上的 FAQ 其中一條 [How much RAM does FreeRTOS use?](https://www.freertos.org/FAQMem.html#RAMUse) 裡面是這樣說的：

*For each task you create, add	64 bytes (includes 4 characters for the task name) + the task stack size.*

*(每一個 Task 會使用 64 byte + stack 的空間)*

事實上不一定是這樣，用 `xTaskCreate()` 建立 Task 之後再用 `xPortGetFreeHeapSize()` 將 FreeRTOS 剩餘可用的記憶體印出可能會發現，stack 只給 512 bytes 的 Task 可能的佔用了 600 甚至是 1000 bytes 以上 ，除了 stack 512 bytes 之外的是什麼呢?

往 `xTaskCreate()` 裡面找，找到這段程式

    pxStack = ( StackType_t * ) pvPortMalloc( ( ( ( size_t ) usStackDepth ) * sizeof( StackType_t ) ) );

這段程式就是上面提到的 `sizeof(StackType_t)` 問題，但這裡還不是造成 Task 消耗額外記憶體的部分，接著往下看到這段

    pxNewTCB = ( TCB_t * ) pvPortMalloc( sizeof( TCB_t ) );

有些 IDE 可以即時顯示 `sizeof(TCB_t)` 的值，一般來說這個值應該在 100 ~ 200 bytes 附近，`TCB_t` 是 FreeRTOS 的 Task control block，用於儲存 Task 執行時期狀態，所以實際大小還與 `FreeRTOSConfig.h` 內的設定有關，開啟越多功能 `TCB_t` 就會使用越多的空間。

但是如果看到 `sizeof(TCB_t)` 的值是 1000 多甚至 2000 多 bytes 的狀況，那大概是中了你正在使用的 FreeRTOS port 的標，FreeRTOS 需要 porting 成對應使用的微控制器的版本，每個 porting 可能會依據該微控制器硬體規格來修改 `FreeRTOSConfig.h` 內的預設值，而你正在使用的 port 將 `configUSE_NEWLIB_REENTRANT` 的預設值設為 1 了。

在新版 FreeRTOS 中的 `TCB_t` 中有一個 `struct	_reent xNewLib_reent`，這個結構是用來讓 FreeRTOS 可以簡單的支援 **Newlib** 這個函式庫，可是對於較低階的微控制器 (如 ARM Cortex-M0 系) 在記憶體較小的情況下，每建一個 Task 就要額外消耗 1KB 是非常大的開銷，在沒有要使用 **Newlib** 的狀況下可以將 `FreeRTOSConfig.h` 內的設定改為 0。

    #define configUSE_NEWLIB_REENTRANT        0

這樣應該就可以看到 `sizeof(TCB_t)` 的值大幅減少了。

### 另外關於 Newlib

即便是有使用 **Newlib** 的需求，只給單一個 Task 操作，再與其餘 Task 用 Queue 傳遞資訊會是更佳的做法。

若真的有兩個 Task 都需要使用 **Newlib** 的需求，那就只給那兩個 Task 使用，只要確保使用前將 `_impure_ptr` 指標指向對應 Task 所屬的 `_reent` 即可，並避免在使用中途進 Context Switch，而若是想把這個功能做進 Context Switch 時期，這裡提供一個不需要大幅修改 FreeRTOS 的方法，只需要修改 `tasks.c` 裡面的 `TCB_t`、`xTaskCreate()`、`vTaskSwitchContext()` 這三部分，首先將 `TCB_t` 的 `_reent` 改為指標，在 `xTaskCreate()` 增加判斷參數 `pvParameters` 有沒有內含 `_reent` 指標，有就將其放入此 Task 的 `TCB_t`，而在 `vTaskSwitchContext()` 中則要先判斷是否有 `_reent` 指標接著在將 `_impure_ptr` 指向該 `_reent`。