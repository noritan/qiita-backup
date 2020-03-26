---
title: EventQueueを使ってみたよ (7)
tags: mbed-os PSoC6 EventQueue
author: noritan_org
slide: false
---
# EventQueueを使ってみたよ (7)

[前回の記事][(6)]では、[EventQueue]について[The EventQueue API] Tutorialで優先順位付けまで読んでみました。
今回は、イベントのメモリ領域について考えます。

## 題材について

この記事で題材として取り上げたのは、[The EventQueue API]という[Tutorial]です。

## イベントキューのメモリプール（EventQueue memory pool）

> When you create an instance of the EventQueue, you specify a fixed size for its memory.

> EventQueueのインスタンスを作る時に、固定された大きさのメモリ領域を設定します。

> Because allocating from the general purpose heap is not IRQ safe, the EventQueue allocates this fixed size block of memory during its creation.

> 汎用ヒープ領域からメモリを確保することはIRQから見て安全ではないので、EventQueueはEventQueueインスタンスが作成されるときに固定された大きさのメモリブロックを確保します。

> Although the EventQueue memory size is fixed, the Eventqueue supports various sized events.

> EventQueueのメモリ領域は固定された大きさですが、イベントキューはさまざまな大きさのイベントを扱います。

> Various sized events introduce fragmentation to the memory region.

> さまざまな大きさのイベントは、メモリ領域の分断を引き起こします。

> This fragmentation makes it difficult to determine how many more events the EventQueue can dispatch.

> メモリの分断が起こると、EventoQueueがあとどれくらいの数のイベントを受け付けることができるかを判断することが難しくなります。

> The EventQueue may be able to dispatch many small events, but fragmentation may prevent it from allocating one large event.

> EventQueueは、小さなイベントを受け付けることができるかもしれませんが、メモリの分断により大きなサイズのイベントを確保することができなくなるかもしれません。

EventQueueは、イベントと呼ばれる「関数と引数へのポインタ」をキューに入れて管理します。
そのため、イベントが占めるメモリはイベントによって様々な大きさになります。
EventQueueは、自身が確保しているメモリ領域をイベントに割り振って処理しているので、メモリの分断が発生して使用できない領域が発生してしまう可能性があります。

### イベントの数を計算する（Calculating the number of events）

> If your project only uses fix-sized events, you can use a counter that tracks the number of events the EventQueue has dispatched.

> プロジェクトで使用するイベントが固定サイズのメモリだけを使用するのであれば、EventQueueが受け付けたイベントの数を管理するカウンタを使うことができます。

> If your projects uses variable-sized events, you can calculate the number of available events of a specific size because successfully allocated memory is never fragmented further.

> プロジェクトが可変サイズのイベントを使用するのであれば、ある特定の大きさの使用可能イベントの数を計算することができます。これは確保に成功したメモリが決して分断されることがないためです。

> However, untouched space can service any event that fits, which complicates such a calculation.

> しかしながら、手付かずの領域は大きさが合えばどんなイベントにも使用できる反面、イベント数の計算を困難にします。

かなりややこしい話になってきました。
ここでコードが紹介されているので、コードを見ながら考えます。

```cpp
EventQueue queue(8*sizeof(int)); // 8 words of storage
queue.call(func, 1);       // requires 2 words of storage
queue.call(func, 1, 2, 3); // requires 4 words of storage
// after this we have 2 words of storage left

queue.dispatch(); // free all pending events

queue.call(func, 1, 2, 3); // requires 4 words of storage
queue.call(func, 1, 2, 3); // fails
// remaining storage has been fragmented into 2 events with 2 words
// of storage, no space is left for a 4 word event even though 4 bytes
// exist in the memory region
```

ここで作成されているEventQueueインスタンス`queue`は、8ワードのメモリを確保しています。
その後`queue.call()`メソッドで二つのイベントが登録されます。
一つ目のイベントは、関数名`func`と引数が一つ指定されているので、2ワードのメモリが必要です。
二つ目のイベントは、関数名`func`と引数が三つ指定されているので、4ワードのメモリが必要です。

これらのイベントに確保されたメモリ領域は、`queue.dispatch()`で解放されますが、2ワードと4ワードの区切りは保存されます。そのため、続く`queu.call()`メソッドで4ワードの領域を二つ確保しようとするとメモリが不足してエラーを起こすという事のようです。

で、このコードを実際に実行しようとしましたが、思ったようには動かないという事がわかりました。

* EventQueueのコンストラクタに渡すべきメモリサイズは、もっと大きい。

イベントが必要とする最小のメモリ領域は、`EventQueue.h`で定義されている`EVENTS_EVENT_SIZE`ですが、これには関数と引数の情報だけでなく管理情報も含まれています。
`8*sizeof(int)`では、一つもイベントが入れられません。

* `queue.dispatch()`では、イベントの実行が終わっても戻ってこない。

`queue.dispatch()`は、引数に-1を与えたのと同じことになり、永遠にイベントを取り出して実行するという動作を繰り返します。
したがって、制御が戻ってきません。
ここでは、`queue.dispatch(0)`を使って、キューが空になったら戻ってくるようにしなくてはなりません。

* キューにイベントが入らない場合でも、エラーが発生しない。

`queue.call()`メソッドでイベントが入れられなかった場合、返り値にイベントIDの代わりにゼロが返ってきます。
それ以外にエラーメッセージが表示されるわけでもないので、この返り値を監視する必要があります。

という事で、プログラムを書き換えました。

```cpp
#include "mbed.h"

void func(int a, ...) {
    printf("func(%d)\r\n", a);
}

int main() {
    EventQueue queue(2 * EVENTS_EVENT_SIZE); // two events of storage
    printf("Start\r\n");
    printf("call(1)=%d\r\n", queue.call(func, 1));       // requires 2 words of storage
    printf("call(2)=%d\r\n", queue.call(func, 2, 2, 2)); // requires 4 words of storage
    // after this we have 2 words of storage left

    queue.dispatch(0); // free all pending events

    printf("call(3)=%d\r\n", queue.call(func, 3, 3, 3)); // requires 4 words of storage
    printf("call(4)=%d\r\n", queue.call(func, 4, 4, 4)); // fails
    // remaining storage has been fragmented into 2 events with 2 words
    // of storage, no space is left for a 4 word event even though 4 bytes
    // exist in the memory region
    printf("call(5)=%d\r\n", queue.call(func, 5)); // requires 2 words of storage
    // still there is a 2 words of storage

    queue.dispatch(0); // free all pending events
}
```

まず、原文には`func()`の定義がありませんでしたので、関数を定義しました。
この関数は、引数を一個以上持つ関数で、どのイベントに起因する呼び出しであるかがわかるように、第一引数にシーケンス番号を入れて、関数内で表示させるようにしました。

EventQueueのインスタンス`queue`を作成するときに渡すメモリサイズは、`EVENTS_EVENT_SIZE`の２倍として、イベントが二つ入れられるようにしています。
そして、`queue.call()`でイベントを登録するときには、その返り値を表示させるようにしました。

`queue.dispatch()`には、イベントを実行したあと制御が戻ってくるようにゼロを与えています。

原文にあった４つのイベントに加えて、五つ目のイベントを加えて、サイズが合えばキューに入れられるという事を示し、最後にキューに残ったすべてのイベントを実行します。

実行した結果は、以下の通りです。

```plaintext
Start
call(1)=128
call(2)=172
func(1)
func(2)
call(3)=300
call(4)=0
call(5)=256
func(3)
func(5)
```

１番目と２番目のイベントは、正常にキューに入り、実行されます。
３番目のイベントをキューに入れて４番目のイベントをキューに入れようとしますが、返り値にゼロが返ってきます。
これは、４番目のイベントが受け入れられなかったことを示します。
原因は、メモリが分断されているので４番目のイベントが入らなかったためです。
ところが、引数を少なくした５番目のイベントはキューに入りますし、そのあとの実行もできます。
ここから、イベントを入れる場所はあったけれど、サイズが小さかったために入れられなかったのだとわかります。

次回は、イベントへのメモリ割り当てについてさらに考えます。

## 追記 (2020-02-08)

`EventQueue`インスタンスを作成するときに指定するメモリサイズが、まったく足りていないとgithubにIssueを出したところ、プログラム例が以下のように修正されました。

```cpp
// event size in words: 9 + callback_size + arguments_size (where 9 is internal space for event data)
void func1(int);
void func3(int, int, int);

EventQueue queue(2*(9+4)*sizeof(int)); // 26 words of storage (store two callbacks with three arguments at max)
queue.call(func1, 1);       // requires 11 words of storage (9+2)
queue.call(func3, 1, 2, 3); // requires 13 words of storage (9+4)
// after this we have 2 words of storage left

queue.dispatch(); // free all pending events

queue.call(func, 1, 2, 3); // requires 13 words of storage (9+4)
queue.call(func, 1, 2, 3); // fails
// storage has been fragmented into two events with 11 and 13 words
// of storage, no space is left for an another 13 word event even though two words
// exist in the memory region
```

一つのイベントに必要なメモリ容量は、「9 + callback_size + arguments_size」と定義されました。
したがって、これまで「8*sizeof(int)」とされていたものが「2*(9+4)*sizeof(int)」になりました。
これは、callbackへのポインタと三つの引数へのポインタを持つことの出来るメモリ領域の２倍を表しています。

これで、メモリが分断されるとイベントがキューに入らない状況を再現することができるとされているのですが、`queue.dispatch()`の引数も省略されたままなので問題が発生する箇所までたどり着けません。

また、相変わらず`queue.call()`メソッドの返り値を確認していないので、入ったのか入らないのか区別がつかないという問題はそのままです。

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
