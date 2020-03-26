---
title: EventQueueを使ってみたよ (4)
tags: mbed-os PSoC6 mutex EventQueue
author: noritan_org
slide: false
---
# EventQueueを使ってみたよ (4)

[前回の記事][(3)]では、[EventQueue]について[The EventQueue API] Tutorialでイベントループを使うところまで読んでみましたが、プログラムに問題があることがわかりました。
今回は、問題点について考えます。

## 題材について

この記事で題材として取り上げたのは、[The EventQueue API]という[Tutorial]です。

## イベントループを使うときに考えられる問題

原文には、この見出しはありません。私が付けたものです。

[前回の記事][(3)]からプログラムを再掲します。

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

> The code for `rise_handler` is problematic because it calls `printf` in interrupt context, which is a potentially unsafe operation.

> `rise_handler`のコードには問題があります。割り込みcontextの中から`printf`を呼び出している事で潜在的に安全でない操作を行っているからです。

> Fortunately, this is exactly the kind of problem that event queues can solve.

> 幸いなことに、これはまさしくイベントキューによって解決される種類の問題です。

> We can make the code safe by running `rise_handler` in user context (like we already do with `fall_handler`) by replacing this line:

> このコードは、すでに`fall_handler`でそうしたように、`rise_handler`をユーザcontextで実行することによって安全にすることができます。以下のコードを

```cpp
sw.rise(rise_handler);
```

> with this line:

> このように書き換えます。

```cpp
sw.rise(queue.event(rise_handler));
```

この文書では、「`rise_handler`を割り込みcontextから呼び出すのは安全ではない。」としていますが、なぜ安全ではないかについては述べられていません。
ここで少し考えてみます。

安全ではないとされていたのは`printf`の呼び出しです。
`printf`は、複数のスレッドから呼び出される可能性があり、しかも出力する表示に乱れがあってはならないために、一度に複数の要求を受け付けないようにしなくてはなりません。
このような場合に使われるのが、セマフォ(semaphore)とかミューテックス(mutex)と呼ばれる仕組みです。
この仕組みを使うと、あるスレッドで`printf`を実行している間は、他のスレッドで`printf`を実行できないようにすることができます。
`printf`の処理をする立場から考えると安全です。

ところが、この`printf`をISRで使ってしまうと問題が発生します。
もし、他のスレッドで`printf`が実行中であった場合、ISRの処理は`printf`を呼び出したところで、現在実行中の`printf`の処理が終わるまで待たされます。
しかし、`printf`を実行中の他のスレッドは割り込みが発生した時点で停止しているため、`printf`の処理は永遠に終わりません。
こうして、デッドロックの状態に陥ってしまうのです。

この文書で作成されたプログラム例でも、ユーザcontextで`fall_handler`が`printf`を実行中に、`rise_handler`がISRから呼び出されるとデッドロックが発生するはずです。
ボタンを素早く押し続けていると、そういった状況になる可能性があります。

こういった問題が発生しないように、現在の[Mbed OS]では、割り込みcontextでの`Mutex`の操作が禁止されたのだろうと考えられます。


このプログラムには、まだ問題がありそうですが、次回に続きます。

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
