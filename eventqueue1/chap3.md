---
title: EventQueueを使ってみたよ (3)
tags: mbed-os PSoC6 EventQueue
author: noritan_org
slide: false
---
# EventQueueを使ってみたよ (3)

[前回の記事][(2)]では、[EventQueue]について[The EventQueue API] Tutorialでイベントループを作るところまで読んでみました。
今回は、イベントループを使います。

## 題材について

この記事で題材として取り上げたのは、[The EventQueue API]という[Tutorial]です。

## イベントループを使う(Using the event loop)

> Once you start the event loop, it can post events.

> イベントループが走り始めたら、イベントを投稿（post）できます。

> Let's consider an example of a program that attaches two interrupt handlers for an InterruptIn object, using the InterruptIn `rise` and `fall` functions.

> InterruptInオブジェクトの`rise`関数と`fall`関数を使って、ひとつのInterruptInオブジェクトにふたつの割り込みハンドラを付けたプログラムの例を考えてみます。

> The `rise` handler will run in interrupt context, and the `fall` handler will run in user context (more specifically, in the context of the event loop's thread).

> `rise`ハンドラは、割り込みcontextで実行され、`fall`ハンドラはユーザcontext（もっと具体的に言うとイベントループスレッドのcontext）で実行されます。

> The full code for the example can be found below:

> このプログラム例の全コードは、以下の通りです。

```cpp
#include "mbed.h"

DigitalOut led1(LED1);
InterruptIn sw(USER_BUTTON, PullUp);
EventQueue queue(32 * EVENTS_EVENT_SIZE);
Thread thread;

void rise_handler(void) {
    printf("rise_handler in context %p\r\n", ThisThread::get_id());
    // Toggle LED
    led1 = !led1;
}

void fall_handler(void) {
    printf("fall_handler in context %p\r\n", ThisThread::get_id());
    // Toggle LED
    led1 = !led1;
}

int main() {
    // Start the event queue
    thread.start(callback(&queue, &EventQueue::dispatch_forever));
    printf("Starting in context %p\r\n", ThisThread::get_id());
    // The 'rise' handler will execute in IRQ context
    sw.rise(rise_handler);
    // The 'fall' handler will execute in the context of thread 't'
    sw.fall(queue.event(fall_handler));
}
```

いよいよ、本格的なコードが出てきましたが、いくつか問題点がありました。

このプログラム例では、押しボタンで操作を行いますが、[PSoC 6]の評価ボードでは`SW2`端子が定義されていません。
代わりに`USER_BUTTON`を使いました。
また、この端子にはプルアップ抵抗が必要なので、`sw`インスタンス生成時の引数に`PullUp`を加えました。

このプログラム例では、`Thread::gettid()`という関数で現在実行中のスレッドIDを取り出しています。
ところが、この関数は`deprecated`扱いになっています。
ここでは、代わりに`ThisThread::get_id()`を使用します。

> The above code executes two handler functions (`rise_handler` and `fall_handler`) in two different contexts:

> このコードでは、二つのハンドラ関数、`rise_handler`と`fall_handler`を二つの異なるcontextで実行しています。

> 1) In interrupt context when a rising edge is detected on `SW2` (`rise_handler`).

> 1) `SW2`で立ち上がりエッジが検出されたら割り込みcontextで`rise_handler`が実行されます。

> 2) In the context of the event loop's thread function when a falling edge is detected on `SW2` (`fall_handler`).
> `queue.event()` is called with `fall_handler` as an argument to specify that `fall_handler` will run in user context instead of interrupt context.

> 2) `SW2`で立ち下がりエッジが検出されたらイベントループのスレッドのcontextで`fall_handler`が実行されます。
> `fall_handler`を引数に持った`queue.event()`が呼び出され、`fall_handler`が割り込みcontextの代わりにユーザcontextで実行されます。

これで、押しボタンの「押す」と「放す」の動作に割り込みが関連付けられ、それぞれ別のcontextで実行されるようになりました。

`queue.event()`は、引数に指定したcallbackをキューに入れる動作をするcallbackを返します。
ややこしいな。

> This is the output of the above program on an FRDM-K64F board.
> We reset the board and pressed the SW2 button twice:

> このプログラムをFRDM-K64Fボードで実行した結果がこの出力です。
> ボードをリセットした後、SW2ボタンを二回押しました。

```plaintext
Starting in context 20001fe0
fall_handler in context 20000b1c
rise_handler in context 00000000
fall_handler in context 20000b1c
rise_handler in context 00000000
```

> The program starts in the context of the thread that runs the `main` function (`20001fe0`).

> `main`関数が走るスレッドのcontext(`20001fe0`)でプログラムが走り始めます。

> When the user presses `SW2`, `fall_handler` is automatically queued in the event queue, and it runs later in the context of thread `t` (`20000b1c`).

> ユーザが`SW2`を押したら、`fall_handler`が自動的にイベントキューに入り、後でスレッド`t`のcontext(`2000b1c`)で実行されます。

> When the user releases the button, `rise_handler` is executed immediately, and it displays `00000000`, indicating that the code ran in interrupt context.

> ユーザがボタンを離したら、`rise_handler`が直ちに実行されます。
> このとき`00000000`が表示されていますが、これは割り込みcontextでコードが実行されたことを示します。

という事なんですが、[PSoC 6]で実行してみたところ、こんな結果にはなりませんでした。

```plaintext
Starting in context 080047ac
fall_handler in context 080051c0


++ MbedOS Error Info ++
Error Status: 0x80010133 Code: 307 Module: 1
Error Message: Mutex: 0x8003C6C, Not allowed in ISR context
Location: 0x100127C5
Error Value: 0x8003C6C
Current Thread: rtx_idle Id: 0x80047F0 Entry: 0x10010359 StackSize: 0x300 StackMem: 0x8004878 SP: 0x80476F4
For more info, visit: https://mbed.com/s/error?error=0x80010133&tgt=CY8CKIT_062_WIFI_BT
-- MbedOS Error Info --
```

ボタンを押した後の`fall_handler`は走ったようなのですが、ボタンを放した後、エラーで停止してしまい、`LED1`が ーーーー・・・・ というパターンで点滅を始めました。
メッセージを読むと、ISR contextでMutexを使用したのが気に入らなかったようです。

実は、このプログラムに問題があることは、この後の部分で記述されているのですが、おそらくこの文書を書いた当時は、プログラムがエラーで停止するような動きにはならなかったのでしょう。

どういう問題があるのかについては、次回に続きます。

## 追記 (2020-02-08)

このISRでMutexを呼び出したら止まってしまうという問題をgithubのIssueに提起したところ、「printfを使うからまずいんでしょ。`UnbufferedSerial::write`を使えばいいんじゃない？」という解決方法が提示されました。

```cpp
// It's safe to use UnbufferedSerial in ISR context
UnbufferedSerial console(USBTX, USBRX);

void rise_handler(void) {
    char buf[64];
    sprintf(buf, "rise_handler in context %p\n", ThisThread::get_id());
    console.write(buf, strlen(buf));
    // Toggle LED
    led1 = !led1;
}
```

が、`UnbufferedSerial`って、どこにあるのさ？


## 追記 (2020-02-09)

`UnbufferedSerial` がどこにあるか見つけられないので、代わりになるクラスがないかと探したところ、`RawSerial`というのを見つけました。

```cpp
// It's safe to use UnbufferedSerial in ISR context
RawSerial console(USBTX, USBRX);

void rise_handler(void) {
    console.printf("rise_handler in context %p\n", ThisThread::get_id());
    // Toggle LED
    led1 = !led1;
}
```

コンソールには、以下のように表示されました。

```plaintext
fall_handler in context 08005380
rise_handler in context 080049b0
fall_handler in context 080053rise_handler in context 08005380
80
fall_handler in context rise_handler in context 08005380
08005380
fall_handler in context 08005380
rise_handler in context 080049b0
fall_handler in context 08005380
```

排他処理が省略されているので、乱れまくりになりました。


## 関連サイト
* [Mbed OSのページ][Mbed OS]
* [EventQueueのTutorial][The EventQueue API]
* [Mbed対応Cypress製品のページ][mbed cypress]

## 関連記事
* [EventQueueを使ってみたよ (1)][(1)]
* [EventQueueを使ってみたよ (2)][(2)]
* [EventQueueを使ってみたよ (3)][(3)]
* [EventQueueを使ってみたよ (4)][(4)]
* [EventQueueを使ってみたよ (5)][(5)]
* [EventQueueを使ってみたよ (6)][(6)]
* [EventQueueを使ってみたよ (7)][(7)]
* [EventQueueを使ってみたよ (8)][(8)]
* [EventQueueを使ってみたよ (9)][(9)]

[(1)]:./chap1.md
[(2)]:./chap2.md
[(3)]:./chap3.md
[(4)]:./chap4.md
[(5)]:./chap5.md
[(6)]:./chap6.md
[(7)]:./chap7.md
[(8)]:./chap8.md
[(9)]:./chap9.md
[PSoC 6]:https://www.cypress.com/psoc6
[Mbed OS]:https://www.mbed.com/platform/mbed-os/
[mbed cypress]:https://os.mbed.com/teams/Cypress/
[EventQueue]:https://os.mbed.com/docs/mbed-os/v5.15/apis/eventqueue.html
[The EventQueue API]:https://os.mbed.com/docs/mbed-os/v5.15/tutorials/the-eventqueue-api.html
[Tutorial]:https://os.mbed.com/docs/mbed-os/v5.15/tutorials/index.html
