---
title: EventQueueを使ってみたよ (6)
tags: mbed-os PSoC6 EventQueue priority
author: noritan_org
slide: false
---
# EventQueueを使ってみたよ (6)

[前回の記事][(5)]では、[EventQueue]について[The EventQueue API] Tutorialでイベントループを使うところまで読んでみました。
今回は、優先順位について考えます。

## 題材について

この記事で題材として取り上げたのは、[The EventQueue API]という[Tutorial]です。

## 優先順位付け（Prioritization）

> The EventQueue has no concept of event priority.

> EventQueueは、イベントの優先順位という概念がありません。

> If you schedule events to run at the same time, the order in which the events run relative to one another is undefined.

> 複数のイベントが同時に実行されるようにスケジュールされたとき、それぞれのイベント同士の順序は定義されていません。

> The EventQueue only schedules events based on time.

> EventQueueは、イベントを時間に基づいてスケジューリングをするだけです。

> If you want to separate your events into different priorities, you must instantiate an EventQueue for each priority.

> 複数のイベントに異なる優先順位を設定したいときには、それぞれの優先順位について別々のEventQueueオブジェクトのインスタンスを作成しなくてはなりません。

> You must appropriately set the priority of the thread dispatching each EventQueue instance.

> それぞれのEventQueueインスタンスを扱うスレッドの優先順位を適切に設定しなくてはなりません。

イベントキューは、単純なキューとして働くので、キューに入ったイベント同士に優先順位をつけることはできません。
そういった動作が必要な時は、別々のイベントキューを作成し、それぞれを別のスレッドに割り当てて、スレッドの優先順位によってイベントの優先順位を決定します。

### コードを書いてみた

優先順位付けの項目は、説明文だけだったので、独自にプログラムを作成してみました。

```cpp
#include "mbed.h"

EventQueue queue1(32 * EVENTS_EVENT_SIZE);
Thread thread1(osPriorityNormal);
EventQueue queue2(32 * EVENTS_EVENT_SIZE);
Thread thread2(osPriorityNormal);

void thread1Task(void) {
    printf("  thread1: start %p at %lld\r\n", ThisThread::get_id(), Kernel::get_ms_count());
    CyDelay(2000);
    printf("  thread1: end %p at %lld\r\n", ThisThread::get_id(), Kernel::get_ms_count());
}

void thread2Task(void) {
    printf("thread2: start %p at %lld\r\n", ThisThread::get_id(),  Kernel::get_ms_count());
    CyDelay(3000);
    printf("thread2: end %p at %lld\r\n", ThisThread::get_id(),  Kernel::get_ms_count());
}

int main() {
    // Start the event queue
    thread1.start(callback(&queue1, &EventQueue::dispatch_forever));
    thread2.start(callback(&queue2, &EventQueue::dispatch_forever));
    printf("Starting in context %p at %lld\r\n", ThisThread::get_id(), Kernel::get_ms_count());
    // Queue two tasks into queue2
    queue2.call(thread2Task);
    queue2.call(thread2Task);
    // Queue three tasks into queue1
    queue1.call(thread1Task);
    queue1.call(thread1Task);
    queue1.call(thread1Task);
}
```

ふたつのイベントキュー`queue1`と`queue2`を作成して、それぞれをスレッド`thread1`と`thread2`で走らせます。
まずは、どちらのスレッドも同じ優先順位`osPriorityNormal`で実行します。
それぞれのスレッドで実行するイベントは、`thread1Task`と`thread2Task`で、`CyDelay()`関数を使って長い時間を要するタスクを模擬しています。

`queue1`には2秒間CPUを使う`thread1Task`を三つ、`queue2`には3秒間CPUを使う`thread2Task`を二つ入れています。
実行結果は、以下の通りです。

```plaintext
Starting in context 080043e0 at 0
  thread1: start 08004e14 at 0
thread2: start 08004ed4 at 6
  thread1: end 08004e14 at 4152
  thread1: start 08004e14 at 4152
thread2: end 08004ed4 at 6239
thread2: start 08004ed4 at 6239
  thread1: end 08004e14 at 8315
  thread1: start 08004e14 at 8315
  thread1: end 08004e14 at 12483
thread2: end 08004ed4 at 12485
```

まず、`thread1`と`thread2`のタスクが6ミリ秒の差で実行を開始します。
最初の`thread1`のタスクは4152ミリ秒後に終了します。
CPUを独占できれば2秒で終わるはずですが、およそ2倍の時間を要しています。
このことから、`thread1`と`thread2`のタスクが、ほぼ同等に実行されていたことがわかります。
その後も、`thread1`と`thread2`は、CPUを分け合いながら実行されていることがわかります。

次に、`thread1`の優先順位を`thread2`よりも高くしてみます。

```cpp
EventQueue queue1(32 * EVENTS_EVENT_SIZE);
Thread thread1(osPriorityAboveNormal);
EventQueue queue2(32 * EVENTS_EVENT_SIZE);
Thread thread2(osPriorityNormal);
```

実行結果は、こうなりました。

```plaintext
Starting in context 080043e0 at 0
  thread1: start 08004e14 at 0
  thread1: end 08004e14 at 2076
  thread1: start 08004e14 at 2076
  thread1: end 08004e14 at 4150
  thread1: start 08004e14 at 4150
  thread1: end 08004e14 at 6225
thread2: start 08004ed4 at 6225
thread2: end 08004ed4 at 9340
thread2: start 08004ed4 at 9340
thread2: end 08004ed4 at 12463
```

`thread1`の優先順位が高くなったことで、`thread2`は`thread1`のイベントがすべて終わるまで実行されません。
その代わり、全体の終了時刻は、少しだけ早くなっています。
これは、タスクの切り替えによるオーバヘッドが少なくなったためと考えられます。

次回は、イベントへのメモリ割り当てについて考えます。

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
