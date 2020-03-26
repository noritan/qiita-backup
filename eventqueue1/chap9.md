---
title: EventQueueを使ってみたよ (9)
tags: mbed-os PSoC6 EventQueue
author: noritan_org
slide: false
---
[前回の記事][(8)]では、[EventQueue]について[The EventQueue API] Tutorialでメモリの分断まで読んでみました。今回は、スタティックなEventQueueについて考えます。

## 題材について

この記事で題材として取り上げたのは、[The EventQueue API]という[Tutorial]です。

## 静的なEventQueue（Static EventQueue）

> The EventQueue API provides a mechanism for creating a static queue, a queue that doesn't use any dynamic memory allocation and accepts only user-allocated events.

> EventQueueのAPIでは、静的なキューを作成する仕組みを提供しています。静的なキューは、動的なメモリ割り当てを使わず、ユーザが割り当てたイベントのみを使用します。

> After you create a static queue (by passing zero as `size` to its constructor), you can post `UserAllocatedEvent` to it.

> コンストラクタの引数`size`にゼロを与えて静的なキューを作成した後、`UserAllocatedEvent`をキューに入れることができます。

> Using static EventQueue combined with UserAllocatedEvent ensures no dynamic memory allocation will take place during queue creation and events posting and dispatching.

> UserAllocatedEventと静的なEventQueueを組み合わせることで、キューの作成とイベントの投入および実行に動的なメモリ割り当てを使用しないことが確約されます。

> You can also declare queues and events as static objects (static in the C++ sense), and then memory for them will be reserved at compile time:

> また、キューとイベントを静的なオブジェクト（C++でいうところのstatic）として宣言することもできるので、それらのメモリはコンパイル時に確保されます。

[前回の記事][(8)]までで使用してきたEventQueueでは、実行時にコンストラクタに引数を与えて、必要なメモリを確保して動作する方式をとっていました。ここで紹介されている静的EventQueueは、コンパイル時にメモリを確保してしまう方式です。

ややこしいことは、いいから、コードを見てみましょう。少し、原文から変更を加えています。

```ccp
/*
* Copyright (c) 2019 ARM Limited. All rights reserved.
* SPDX-License-Identifier: Apache-2.0
* Licensed under the Apache License, Version 2.0 (the License); you may
* not use this file except in compliance with the License.
* You may obtain a copy of the License at
*
* http://www.apache.org/licenses/LICENSE-2.0
*
* Unless required by applicable law or agreed to in writing, software
* distributed under the License is distributed on an AS IS BASIS, WITHOUT
* WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
* See the License for the specific language governing permissions and
* limitations under the License.
*/

#include "mbed.h"
```

いちおう、クレジットも入れておきます。

```cpp
// Creates static event queue
static EventQueue queue(0);
```

これまで、EventQueueインスタンスを作成するときには、引数にメモリ領域の大きさを指定していました。静的EventQueueでは、実行時にメモリ割り当てを行わせないようにゼロを与えています。

```cpp
void handler(int count);

// Creates events for later bound
auto event1 = make_user_allocated_event(handler, 1);
auto event2 = make_user_allocated_event(handler, 2);

// Creates event bound to the specified event queue
auto event3 = queue.make_user_allocated_event(handler, 3);
auto event4 = queue.make_user_allocated_event(handler, 4);
```

これまでは、実行時に動的にイベントを作成していましたが、この例ではコンパイル時にイベントを作成します。`make_user_allocated_event()`関数は、EventQueueを選ばない`UserAllocatedEvent`オブジェクトを作成します。一方、`make_user_allocated_event()`メソッドは、特定のEventQueueに紐づいた`UserAllocatedEvent`オブジェクトを作成します。

```cpp
void handler(int count) {
    printf("UserAllocatedEvent = %d at %lld\n", count, Kernel::get_ms_count());
    return;
}
```

それぞれのイベントで実行される関数`handler`では、引数の値を表示して、何番目のイベントが呼び出されたかを示します。ちょっと改造して、時刻情報の表示を追加しました。

```cpp
void post_events(void) 
{
    // Single instance of user allocated event can be posted only once.
    // Event can be posted again if the previous dispatch has finished or event has been canceled.

    // bind & post
    event1.call_on(&queue);

    // event cannot be posted again until dispatched or canceled
    bool post_succeed = event1.try_call();
    assert(!post_succeed);

    queue.cancel(&event1);

    // try to post
    post_succeed = event1.try_call();
    assert(post_succeed);

    // bind & post
    post_succeed = event2.try_call_on(&queue);
    assert(post_succeed);

    // post
    event3.call();

    // post
    event4();
}
```

イベントをキューに入れる方法が示されています。特定のキューを指定しないイベント`event1`では、`event1.call_on()`メソッドで投入すべきキューを引数で指定しています。

一度キューに投入したイベントは、実行またはキャンセルされるまで再投入ができません。`event1.try_call()`メソッドでは、`event1`のキューへの投入を試し、投入できたかどうかを返り値で示します。`event1.call_on()`メソッドでは、この返り値は内部で処理されて外には出てきません。ここでは、`event1.call_on()`メソッドで投入した直後であるため、投入に失敗することを期待しています。

このままでは、イベント`event1`の投入ができないので、いったんキューに入っているイベントを`queue.cancel(&event1)`でキャンセルします。すると、こんどは`event1.try_call()`でイベントを投入できるようになります。

`try_call()`メソッドには、キューを指定することができる`event2.try_call_on(&queue)`メソッドもあります。

イベントの生成時に`queue.make_user_allocated_event()`メソッドを使うと、イベントの投入時にキューの指定を`event3.call()`のように省略できます。メソッドそのものを省略して`event4()`とすることもできます。

```cpp
int main() 
{
    printf("*** start ***\n");
    Thread event_thread;

    // The event can be manually configured for special timing requirements
    // specified in milliseconds
    // Starting delay - 100 msec
    // Delay between each event - 200msec
    event1.delay(100);
    event1.period(200);
    event2.delay(100);
    event2.period(200);
    event3.delay(100);
    event3.period(200);
    event4.delay(100);
    event4.period(200);

    event_thread.start(callback(post_events));

    // Posted events are dispatched in the context of the queue's dispatch function
    queue.dispatch(600);        // Dispatch time - 600msec
    // 600 msec - 3 set of events will be dispatched as period is 200 msec

    event_thread.join();
}
```

四つのイベントには、それぞれ`delay()`メソッドと`period()`メソッドで実行時期と実行周期を設定しています。こうすると、イベントをキューに入れてから100ms、300ms、500ms、というタイミングでイベントが実行されます。

イベントをキューに入れるための関数`post_events`は、`event_thread`という独立したスレッドで実行されます。EventQueueインスタンスは、`queue.dispatch(600)`により600msだけイベントの実行に使われるので、それぞれのイベントは3回ずつ実行されるはずです。はたして実行結果は、こうなりました。

```plaintext
*** start ***
UserAllocatedEvent = 1 at 100
UserAllocatedEvent = 2 at 100
UserAllocatedEvent = 3 at 100
UserAllocatedEvent = 4 at 100
UserAllocatedEvent = 2 at 300
UserAllocatedEvent = 3 at 300
UserAllocatedEvent = 4 at 300
UserAllocatedEvent = 2 at 500
UserAllocatedEvent = 3 at 500
UserAllocatedEvent = 4 at 500
```

`event1`だけが繰り返し実行されていません。

四つのイベントの中で`event1`だけは`queue.cancel(&event1)`でキューから取り除かれて、再度キューに入れられています。試しに`queue.cancel(&event1);`から`assert(post_succeed);`までをコメントアウトしてみたら、`event1`も3回実行されるようになりました。どうやら、`queue.cancel(&event1)`で`period`の値がリセットされてしまったようです。

そこで、`queue.cancel(&event1)`の後、`period`を再設定してみました。

```cpp
    queue.cancel(&event1);
    event1.period(180);
```

振る舞いの違いが分かるように180ms周期にしてみました。すると、`event1`も3回実行されるようになりました。

```plaintext
*** start ***
UserAllocatedEvent = 1 at 100
UserAllocatedEvent = 2 at 100
UserAllocatedEvent = 3 at 100
UserAllocatedEvent = 4 at 100
UserAllocatedEvent = 1 at 280
UserAllocatedEvent = 2 at 300
UserAllocatedEvent = 3 at 300
UserAllocatedEvent = 4 at 300
UserAllocatedEvent = 1 at 460
UserAllocatedEvent = 2 at 500
UserAllocatedEvent = 3 at 500
UserAllocatedEvent = 4 at 500
```

`delay`の値は再設定していないのですが、どうやら、`period`はリセットされるのに、`delay`はリセットされていない様子です。

以上で、Tutorialは、おしまいです。

## 追記 (2020-02-08)

このコード例での「event1のperiodがcancelによってリセットされる」問題もgithubのIssueに提起したところ、「コード例に問題はない。バグを見つけた。」という反応がありました。

つまり、ライブラリに問題があるため、思った通りの動作にならなかったという事ですか？


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
[mbed-events readme]:https://github.com/ARMmbed/mbed-os/blob/master/events/README.md
[equeue library]:https://os.mbed.com/docs/mbed-os/v5.15/mbed-os-api-doxy/_event_queue_8h_source.html
