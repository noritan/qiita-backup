---
title: EventQueueを使ってみたよ (5)
tags: mbed-os PSoC6 EventQueue
author: noritan_org
slide: false
---
[前回の記事][(4)]では、[EventQueue]について[The EventQueue API] Tutorialで割り込みcontextで`printf`を使った場合の問題まで読んでみました。この問題はイベントループで解決できそうですが、プログラムには他にも問題がありそうです。

## 題材について

この記事で題材として取り上げたのは、[The EventQueue API]という[Tutorial]です。

## 処理遅延が気になる場合

原文には、この見出しはありません。私が付けたものです。

[前回の記事][(4)]で安全に改造したプログラムを示します。

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
    // The 'rise' handler will execute in the context of thread 't'
    sw.rise(queue.event(rise_handler));
    // The 'fall' handler will execute in the context of thread 't'
    sw.fall(queue.event(fall_handler));
}
```

> The code is safe now, but we may have introduced another problem: latency.

> これでコードは安全になりましたが、ここでもう一つの問題を提起します。遅延です。

> After the change above, the call to `rise_handler` will be queued, which means that it no longer runs immediately after the interrupt is raised.

> 上記のようにコードを変更すると、`rise_handler`の呼び出しはキューに入ります。これは、割り込みが発生した直後には絶対に`rise_handler`が実行されないことを意味します。

> For this example code, this isn't a problem, but some applications might need the code to respond as fast as possible to an interrupt.

> このプログラム例では、遅延は問題となりませんが、割り込みの発生から可能な限り早く応答するコードが、ある種のアプリケーションでは必要とされます。

イベントキューに入ったイベントは、キューに入った時点では実行されません。いつ実行されるかは、オペレーティングシステムによります。割り込み発生後、即座に反応しなくてはならない場合には、キューに入ってから実際に実行されるまでの遅延が問題になってくるというわけです。

> Let's assume that `rise_handler` must toggle the LED as quickly as possible in response to the user's action on SW2.

> SW2へのユーザの操作に応答して`rise_handler`が可能な限り早くLEDをトグルする場合を考えてみます。

> To do that, it must run in interrupt context.

> このような場合、`rise_handler`は割り込みcontextで実行しなくてはなりません。

> However, `rise_handler` still needs to print a message indicating that the handler was called; that's problematic because it's not safe to call `printf` from an interrupt context.

> しかしながら、ハンドラが呼び出されたことを示すために`rise_handler`がメッセージを表示するのも、いまだ必要です。割り込みcontextから`printf`を呼び出すのは安全ではないので問題になります。

メッセージの表示は、人間向けの処理だからキューに入れてしまって構わないのだけれど、LEDの応答は、割り込みのあと速やかに実行したいという状況です。

> The solution is to split `rise_handler` into two parts: the time critical part will run in interrupt context, while the non-critical part (displaying the message) will run in user context.

> 解決法は、`rise_handler`を二つの部分に分けることです。時間に厳しい処理は割り込みcontextで実行し、時間に厳しくないメッセージ表示のような処理はユーザcontextで実行します。

> This is easily doable using `queue.call`:

> この記述は、`queue.call`を使って簡単に実現できます。

```cpp
void rise_handler_user_context(void) {
    printf("rise_handler_user_context in context %p\r\n", ThisThread::get_id());
}

void rise_handler(void) {
    // Execute the time critical part first
    led1 = !led1;
    // The rest can execute later in user context (and can contain code that's not interrupt safe).
    // We use the 'queue.call' function to add an event (the call to 'rise_handler_user_context') to the queue.
    queue.call(rise_handler_user_context);
}
```

急ぎの仕事と急ぎでない仕事を分離してしまえばよいという解決法です。LEDをトグルするという急ぎの仕事は、割り込みcontextでそのまま処理しますが、メッセージを表示するという急ぎでない仕事`rise_handler_user_context()`は、イベントキューに入れてあとから処理してしまいます。

この時、`main()`の処理を元に戻すのをお忘れなく。

```cpp
    // The 'rise' handler will execute in IRQ context
    sw.rise(rise_handler);
    // The 'fall' handler will execute in the context of thread 't'
    sw.fall(queue.event(fall_handler));
```

> After replacing the code for `rise_handler` as above, the output of our example becomes:

> `rise_handler`のコードを上のように書き換えると、プログラム例の出力は以下のようになります。

```plaintext
Starting in context 0x20002c50
fall_handler in context 0x20002c90
rise_handler_user_context in context 0x20002c90
fall_handler in context 0x20002c90
rise_handler_user_context in context 0x20002c90
```

> The scenario above (splitting an interrupt handler's code into time critical code and non-time critical code) is another common pattern that you can easily implement with event queues; queuing code that's not interrupt safe is not the only thing you can use event queues for.

> 割り込みハンドラのコードを時間に厳しいコードと時間に厳しくないコードに分離するという上記の考え方は、イベントキューで簡単に実装することができるもう一つの一般的な手順です。割り込みに対して安全でないコードをキューに入れることは、イベントキューを使用する唯一の動機というわけではありません。

> Any kind of code can be queued and deferred for later execution.

> あらゆる種類のコードをキューに入れることができ、先送りされて後で実行されます。

> We used `InterruptIn` for the example above, but the same kind of code can be used with any `attach()`-like functions in the SDK.

> このプログラム例では、`InterruptIn`を使ってきましたが、SDKにある`attach()`系の関数で同じ種類のコードが使用できます。

> Examples include `Serial::attach()`, `Ticker::attach()`, `Ticker::attach_us()`, `Timeout::attach()`.

> たとえば、`Serial::attach()`、`Ticker::attach()`、`Ticker::attach_us()`、`Timeout::attach()`が含まれます。

ここで`attach()`系（`attach()`-like）と呼ばれているのは、割り込みが発生したときに呼び出されるcallback関数を登録する関数の事です。`InterruptIn`オブジェクトでは`rise()`と`fall()`で割り込み発生時に実行するcallbackを登録していましたが、他のオブジェクトでは`attach()`メソッドでcallbackを登録しています。

次回は、優先順位について考えます。

## 関連サイト
[Mbed OSのページ][Mbed OS]
[EventQueueのTutorial][The EventQueue API]
[Mbed対応Cypress製品のページ][mbed cypress]

## 関連記事
[EventQueueを使ってみたよ (1)][(1)]
[EventQueueを使ってみたよ (2)][(2)]
[EventQueueを使ってみたよ (3)][(3)]
[EventQueueを使ってみたよ (4)][(4)]
[EventQueueを使ってみたよ (5)][(5)]
[EventQueueを使ってみたよ (6)][(6)]
[EventQueueを使ってみたよ (7)][(7)]
[EventQueueを使ってみたよ (8)][(8)]
[EventQueueを使ってみたよ (9)][(9)]

[(1)]:https://qiita.com/noritan_org/items/89406171ea7bcef2a665
[(2)]:https://qiita.com/noritan_org/items/ff72ae6a4398ba6d3432
[(3)]:https://qiita.com/noritan_org/items/d8333c74fb8d2ef8a8de
[(4)]:https://qiita.com/noritan_org/items/65d579f722002ea12a6c
[(5)]:https://qiita.com/noritan_org/items/172ca6c62fe4b36767d4
[(6)]:https://qiita.com/noritan_org/items/cc4a0ab2c6ff9c0aa5ec
[(7)]:https://qiita.com/noritan_org/items/83d2728811220c2c44ad
[(8)]:https://qiita.com/noritan_org/items/58316099f9ef45bc56bd
[(9)]:https://qiita.com/noritan_org/items/fa35cc2e07c1841f5eb2
[PSoC 6]:https://www.cypress.com/psoc6
[Mbed OS]:https://www.mbed.com/platform/mbed-os/
[mbed cypress]:https://os.mbed.com/teams/Cypress/
[EventQueue]:https://os.mbed.com/docs/mbed-os/v5.15/apis/eventqueue.html
[The EventQueue API]:https://os.mbed.com/docs/mbed-os/v5.15/tutorials/the-eventqueue-api.html
[Tutorial]:https://os.mbed.com/docs/mbed-os/v5.15/tutorials/index.html
