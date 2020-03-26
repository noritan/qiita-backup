# EventQueueを使ってみたよ (1)
---
title: EventQueueを使ってみたよ (1)
tags: mbed-os PSoC6 EventQueue
author: noritan_org
slide: false
---
[PSoC 6]で[Mbed OS]が[使える][mbed cypress]ようになったので、使い始めました。マルチタスクで動くのが便利だね。というわけで、[EventQueue]について調べてみました。

## 題材について

この記事で題材として取り上げたのは、[The EventQueue API]という[Tutorial]です。ここで出てくるプログラム例を試しながら考えてみます。

## 序文

> One of the optional Arm Mbed OS features is an event loop mechanism that you can use to defer the execution of code to a different context.

> Arm Mbed OS の追加機能の一つにイベントループがあり、コードの実行を異なるcontextに先送りできます。

> In particular, a common use of an event loop is to postpone the execution of a code sequence from an interrupt handler to a user context.

> 特に、一連のコードの実行を割り込みハンドラからユーザcontextに先送りするのが、イベントループの一般的な用途です。

contextという言葉が出てきましたが、コードを実行する場面とでも表現すれば良いでしょうか。例えば、割り込み処理ルーチン (Interrupt Service Routine: ISR) というのは、一般的に特権モードで実行されます。そのため、ISRにそのままコードを書くと他のコードよりも強い権限を持ちます。そういった強い権限が必要であればそれでも良いのですが、もっと弱い権限で実行させたい場合もあります。このような場合に、「ユーザcontext」と呼ばれる一般モードで実行させるのが主な目的というわけです。

> This is useful because of the specific constraints of code that runs in an interrupt handler:

> この機能は、割り込みハンドラで実行されるコードに特有の制約がある場合に便利です。

> * The execution of certain functions (notably some functions in the C library) is not safe.

> * いくつかの関数、特にCライブラリのいくつかの関数の実行は、安全ではありません。

> * You cannot use various RTOS objects and functions from an interrupt context.

> * 様々なRTOSオブジェクトや関数は、割り込みcontextから使用することができません。

> * As a general rule, the code needs to finish as fast as possible, to allow other interrupts to be handled.

> * 一般的な規則として、可能な限り早く処理を終わらせて他の割り込みの処理ができるようにするコードが必要です。

ISRに特有な制約が並んでいます。ISRで時間のかかる処理を行うのはご法度で、RTOS特有のオブジェクトや関数を使って他のタスクを待つような処理は避けなくてはなりません。また、ISRに処理を書いたために、デッドロックに陥ってしまうこともあり得ます。このような状況を避けるためにも、ISRで処理せずに先送りするのが得策というわけです。

シングルタスクのプログラムでも同様です。ISRの中ではフラグを立てるだけにして、メインループでフラグを監視して実際の処理を行うようなプログラムが使用されます。

> The event loop offers a solution to these issues in the form of an API that can defer execution of code from the interrupt context to the user context.

> イベントループは、割り込みcontextからユーザcontextへとコードの実行を先送りすることの出来るAPIとしてこれらの問題点に対する解決策を提供します。

> More generally, the event loop can be used anywhere in a program (not necessarily in an interrupt handler) to defer code execution to a different context.

> もっと一般的に、プログラムのどこでも、割り込みハンドラの中でなくても、別のcontextでコードを実行させるためにイベントループを使用することができます。

イベントループの仕組みは、ISRに限らず使用することができます。例えば、I2Cインターフェイスを扱う場合、I2Cハードウェアの操作は、単一のThreadを作って処理させた方が安全です。このような場合に、I2Cインターフェイスの処理を行うためのイベントループがあれば、Threadをまたいで処理をさせることができます。

> In Mbed OS, events are pointers to functions (and optionally function arguments).

> Mbed OSでは、イベントは関数（と有れば引数）へのポインタです。

> An event loop extracts events from a queue and executes them.

> イベントループは、キューからイベントを取り出して実行します。

というわけで、まだまだ序文だけです。

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
