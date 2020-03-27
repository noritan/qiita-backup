---
title: EventQueueを使ってみたよ (8)
tags: mbed-os PSoC6 EventQueue
author: noritan_org
slide: false
---
# EventQueueを使ってみたよ (8)

[前回の記事][(7)]では、[EventQueue]について[The EventQueue API] Tutorialでメモリの分断が起こる可能性があるというところまで読んでみました。
今回は、メモリの分断についてさらに考えます。

## 題材について

この記事で題材として取り上げたのは、[The EventQueue API]という[Tutorial]です。

### メモリの分断に起因する不具合（Failure due to fragmentation）

> The following example would fail because of fragmentation:

> 以下の例では、メモリの分断によって不具合を発生します。

```cpp
EventQueue queue(4*sizeof(int)); // 4 words of storage
queue.call(func);       // requires 1 word of storage
queue.call(func);       // requires 1 word of storage
queue.call(func);       // requires 1 word of storage
queue.call(func);       // requires 1 word of storage
// 0 words of storage remain

queue.dispatch();  // free all pending events
// all memory is free again (4 words) and in one-word chunks

queue.call(func, 1, 2, 3); // requires 4 words of storage, so allocation fails
```

> Four words of storage are free but only for allocations of one word or less.

> 4ワードのメモリ領域が解放されますが、メモリが分断されたままなので、1ワード以下の領域しか確保できません。

> The solution to this failure is to increase the size of your EventQueue.

> この不具合の解決策は、EventQueueのサイズを大きくすることです。

> Having the proper sized EventQueue prevents you from running out of space for events in the future.

> 適切なサイズのEventQueueを取り扱えば、イベントのためのメモリが将来枯渇するのを防ぐことができます。

と、コードが出てきましたが、このコードも変ですので書き直しました。

```cpp
#include "mbed.h"

void func(int a, ...) {
    printf("func(%d)\r\n", a);
}

int main() {
    EventQueue queue(4 * EVENTS_EVENT_SIZE); // four events of storage
    printf("Start\r\n");
    printf("call(1)=%d\r\n", queue.call(func, 1));       // requires 2 words of storage
    printf("call(2)=%d\r\n", queue.call(func, 2));       // requires 2 words of storage
    printf("call(3)=%d\r\n", queue.call(func, 3));       // requires 2 words of storage
    printf("call(4)=%d\r\n", queue.call(func, 4));       // requires 2 words of storage
    // after this we have 2 words of storage left

    queue.dispatch(0); // free all pending events
    // all memory is free again (4 words) and in one-word chunks

    printf("call(5)=%d\r\n", queue.call(func, 5, 5, 5)); // fails

    queue.dispatch(0); // free all pending events
}
```

実行結果は、以下の通りです。

```plaintext
Start
call(1)=256
call(2)=300
call(3)=344
call(4)=388
func(1)
func(2)
func(3)
func(4)
call(5)=0
```

EventQueueには、4イベント分のメモリ領域を確保しました。
そこに2ワードしか必要としないイベントを4本入れてやると、メモリが分断されて、すべてのイベントが2ワードしか受け入れられなくなります。
`queue.dispatch()`でイベントを実行してイベントを開放しても状況は変わりません。
大きなメモリ領域を必要とする5番目のイベントは、キューに入れることができません。

### イベントに関するさらなる情報（More about events）

> This is only a small part of how event queues work in Mbed OS.

> この文書は、Mbed OSでイベントキューがどのように動作するかについてのほんの一部にしか過ぎません。

> The `EventQueue`, `Event` and `UserAllocatedEvent` classes in the `mbed-events` library offer a lot of features that this document does not cover, including calling functions with arguments, queueing functions to be called after a delay or queueing functions to be called periodically.

> `mbed-events`ライブラリの`EventQueue`、`Event`、`UserAllocatedEvent`の各クラスは、多くの機能を提供しています。
> そこには、この文書で述べられていない、引数を伴った関数の呼び出し、遅れて呼び出される関数をキューに入れること、周期的に呼び出される関数をキューに入れること、が含まれています。

> The [README of the `mbed-events` library][mbed-events readme] shows more ways to use events and event queues.

> [`mbed-events`ライブラリのREADME][mbed-events readme]にイベントとイベントキューのさらなる用途が書かれています。

> To see the implementation of the events library, review [the equeue library][equeue library].

> イベントライブラリの実装について知りたいときには、[equeueライブラリ][equeue library]を参照してください。


次回は、スタティックなEventQueueについて考えます。


## 追記 (2020-02-08)

このコード例も、githubにIssueを出した後、修正されました。

```cpp
// event size in words: 9 + callback_size + arguments_size (where 9 is internal space for event data)
void func0();
void func3(int, int, int);

EventQueue queue(4*(9+1)*sizeof(int)); // 40 words of storage
queue.call(func0);       // requires 10 word of storage (9+1)
queue.call(func0);       // requires 10 word of storage (9+1)
queue.call(func0);       // requires 10 word of storage (9+1)
queue.call(func0);       // requires 10 word of storage (9+1)
// 0 words of storage remain

queue.dispatch();  // free all pending events
// all memory is free again (40 words) and in 10-word chunks

queue.call(func3, 1, 2, 3); // requires 13 words of storage (9+4), so allocation fails
```

が、これも`queue.dispatch()`の引数が無く、`queue.call()`の返り値も確認されていないので、何が起こっているかわからないはずです。


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
[mbed-events readme]:https://github.com/ARMmbed/mbed-os/blob/master/events/README.md
[equeue library]:https://os.mbed.com/docs/mbed-os/v5.15/mbed-os-api-doxy/_event_queue_8h_source.html
