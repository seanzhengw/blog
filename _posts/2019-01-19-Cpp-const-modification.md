---
layout: post
title:  "C++ const 修改"
date:   2019-01-19 18:00:00 +0800
categories: cpp
tags: [cpp]
---

在 C++ 程式裡面有時候會有些奇怪的事情會發生，像是具有 const 屬性的變數或物件其實還是可以透過指標修改或呼叫非 const 成員方法

    void test(const int const_int)
    {
        // 這樣就可以得到沒有 const 屬性的指標
        int* int_p = (int*)&const_int;
        *int_p = *int_p * 2;
    }

只是上面這個方法沒什麼用途，因為在呼叫 `test(x)` 的時候是用傳值方式傳遞，所以修改不到 `x`

所以可以改成這樣

    // 將傳入的值 * 2
    void test(const int* const_int_p){
        int* int_p = (int* )const_int_p;
        *int_p = *int_p * 2;
    }

這樣若是呼叫了 `test(&x)`， `x` 的值會被 x2，明明函式參數有 `const` 屬性，但是 `x` 卻被修改了，對於 `test(const int*)` 的使用者來說這就是很奇怪的事情。

同理，帶有 const 屬性的物件也可以用指標方式繞過編譯器檢查，連 reference 也可以這樣做

    class ConstTest
    {
    public:
        void Set(int val)
        {
            mVal = val;
        }

        int Get() const
        {
            return mVal;
        }

    private:
        int mVal;
    };

    // 不論傳入的 ConstTest 實體有沒有 const 屬性，都將其 mVal * 2
    void test(const ConstTest& c){
        ConstTest* p = (ConstTest*)&c;
        p->Set(p->Get() * 2);
    }

另外在某些情況下會因為編譯器優化而失去設計原意，例如這樣

    #include <iostream>

    int main(int argc, char* argv[])
    {
        int i = 0;
        const int const_int = 10;
        int* const_int_p = (int*)&const_int;
        *const_int_p = 20;
        i = const_int;
        std::cout << i << std::endl;
    }

上方的程式看起來是將 `i` 改為 20，但執行這份程式大多數情況下輸出值都是 10，
因為 `const_int` 帶有 `const` 屬性，多數編譯器會將這個值放在常數表中，要用到的時後直接填入其常數值，所以 `i = const_int;` 的部分會被編譯為 `i = 10;`，這部分可以用編譯器提供的組合語言輸出選項查看，以 gcc 為例，以 `g++ -S main.cpp` 可以輸出 main.s 組合語言檔，在組語檔 main.s 裡面還可以看到 `*const_int_p = 20;` 確實有將 20 放進 `const_int` 的位址，也就是說 `const_int` 的內容確實有被修改。

### volatile

有一個情況是將常數宣告為 `volatile const`，這在嵌入式系統中常作為唯獨暫存器的表示方式，因為 `volatile` 告訴編譯器這個常數應該要每次都從記憶體中取出值，於是不能使用常數表優化，試著將上一段程式的 `const int const_int = 10;` 改為 `volatile const int const_int = 10;`，再編譯執行一次程式會看到輸出值為 20。

### 總結

不要修改 `const`。

以上只是一些有趣的玩法，修改常數值違反了程式語意，任何情況下都不應該這樣做。