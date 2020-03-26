---
title: EventQueueを使ってみたよ (2)
tags: mbed-os PSoC6 EventQueue
author: noritan_org
slide: false
---
[EventQueue]について[The EventQueue API] Tutorialの序文を読んでみた[前回の記事][(1)]に続いて、今回は、イベントループのコードを書きます。

## 題材について

この記事で題材として取り上げたのは、[The EventQueue API]という[Tutorial]です。

## イベントループを作る(Creating an event loop)

お待たせしました。コードが出てきますよ。

> You must create and start the event loop manually.

> イベントループは手作業で作成して実行を始めなくてはなりません。

> The simplest way to achieve that is to create a Thread and run the event queue's `dispatch` method in the thread:

> 実現のための一番簡単な方法は、Threadを作成し、そのスレッドでイベントキューの`dispatch`メソッドを実行することです。

手作業(manually)とは書いてありますが、実際にはもちろんプログラムで実行します。
スレッドを作って、`dispatch`を呼び出して、細かいことはいいからコードを見てみましょう。

```cpp
#include "mbed.h"

// Create a queue that can hold a maximum of 32 events
EventQueue queue(32 * EVENTS_EVENT_SIZE);
// Create a thread that'll run the event queue's dispatch function
Thread thread;

int main () {
    // Start the event queue's dispatch thread
    thread.start(callback(&queue, &EventQueue::dispatch_forever));
}
```

まず、`EventQueue`オブジェクトのインスタンス`queue`を作っています。引数には、このインスタンスで使用するキューのバイト数を指定しています。キューの要素一つ当たり`EVENTS_EVENT_SIZE`の大きさが必要なので、それを32倍して32要素分の領域を確保しています。
次に`Thread`オブジェクトの`thread`を作っています。引数を与えてスタックサイズなどを指定できるのですが、ここではデフォルトの値を使用するので、引数はありません。
`main()`の中では、`thread`インスタンスにcallback関数を与えて実行させています。ここで`callback()`と書かれているのは、callback関数を返すtemplateのようで、要は、こういうことを意味しているようです。

```cpp
#include "mbed.h"

// Create a queue that can hold a maximum of 32 events
EventQueue queue(32 * EVENTS_EVENT_SIZE);
// Create a thread that'll run the event queue's dispatch function
Thread thread;

void dispatchTask(void) {
    queue.dispatch_forever();
}

int main () {
    // Start the event queue's dispatch thread
    thread.start(dispatchTask);
}
```

ここで呼び出している`dispatch_forever`メソッドは、キューからイベントを取り出して実行するという手続きを繰り返す無限ループになっています。このループそのものを制御したいときには、別の`dispatch`メソッドも使用できます。


```cpp
void dispatchTask(void) {
    DigitalOut led1(LED1);
    for (;;) {
        queue.dispatch(500);
        led1 = !led1;
    }
}
```

`dispatch`メソッドの引数には、timeoutの時間をミリ秒単位で与えます。このようにすると、イベントの到着が無かった場合には、`dispatch`メソッドを抜けて別の処理、例えばLEDを点滅させるなどの処理を行うことができます。

```cpp
void dispatchTask(void) {
    queue.dispatch(-1);
}
```

また、負の値を与えた場合には、永遠にイベントの到着を待ちます。これは、`dispatch_forever`メソッドと同じ振る舞いになります。

```cpp
void dispatchTask(void) {
    for (;;) {
        queue1.dispatch(0);
        queue2.dispatch(0);
    }
}
```

引数にゼロを与えると、イベントキューにイベントが到着していない場合、すぐに`dispatch`メソッドを抜けてきます。そのため、このように複数のイベント処理を並べる書き方もできます。しかしながら、これではイベントを検出するためのループにCPUを喰われてしまいます。メモリに余裕があれば、それぞれのイベントキューに独立したThreadを与えた方が良いでしょう。

> Note that though this document assumes the presence of a single event loop in the system, there's nothing preventing the programmer from running more than one event loop, simply by following the create/start pattern above for each of them.

> この文書では、システムに単一のイベントループが存在すると仮定していますが、複数のイベントループを実行するのを妨げる理由は全くありません。それぞれのイベントループに対して「作成して・実行する」という手順を踏むだけです。

次は、作ったイベントループを使います。

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
