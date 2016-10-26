---
title: RxJS - Не используйте unsubscribe непосредственно
date: 2016-10-26 00:00:00
tags: [Переводы, Технологии]
description: <a href="/dont_unsubscribe/"><img src="/dont_unsubscribe/thumb-middle-just-dont-do-it.jpg"></a>
---
Что ж, только не отказывайтесь от отписки слишком сильно.

Я часто берусь помогать кому-нибудь с отладкой и вопросами по RxJS коду или выяснению как лучше структурировать приложение, которое состоит из достаточно многих асинхронных частей. Когда я делаю это, я в целом вижу одну и ту же вещь снова и снова - люди хранят тонны и тонны объектов подписок [те, что возвращает метод `subscribe`]. Разработчики неизменно делают 3 HTTP запроса с помощью Observable и хранят 3 объекта подписки, которые они собираются вызвать когда произойдёт какое-то событие [чтобы уничтожить подписку].

Я могу предположить как это могло бы случиться. Люди привыкли использовать `addEventListener` N раз и после этого необходима некоторая уборка когда они должны вызывать `removeEventListener` так же N раз. Кажется естественным делать то же самое с объектами подписки, и по большей части вы правы. Но есть путь лучше. Хранить слишком много объектов подписки - это знак того, что вы управляете своими подписками императивно, и не используете силу Rx.

## Как выглядит императивное управление подписками

Возьмём для примера этот воображаемый компонент (Я намеренно сделал его не-React, не-Angular, а чем-то более общим)

```js
class MyGenericComponent extends SomeFrameworkComponent {
 updateData(data) {
  // здесь обновляем компонент способом, который зависит от используемого фреймворка
 }

 onMount() {
  this.dataSub = this.getData()
   .subscribe(data => this.updateData(data));

  const cancelBtn = this.element.querySelector(‘.cancel-button’);
  const rangeSelector = this.element.querySelector(‘.rangeSelector’);

  this.cancelSub = Observable.fromEvent(cancelBtn, ‘click’)
   .subscribe(() => {
    this.dataSub.unsubscribe();
   });

  this.rangeSub = Observable.fromEvent(rangeSelector, ‘change’)
   .map(e => e.target.value)
   .subscribe((value) => {
    if (+value > 500) {
      this.dataSub.unsubscribe();
    }
   });
 }

 onUnmount() {
  this.dataSub.unsubscribe();
  this.cancelSub.unsubscribe();
  this.rangeSub.unsubscribe();
 }
}
```

В примере выше вы можете увидеть явный вызов `unsubscribe` на трёх объектах подписок, которыми я управляю сам в методе `onUnmount()`. Так же я вызываю `this.dataSub.unsubscribe()` когда кто-то кликнет по кнопке отмены в строке #12, и ещё раз в строке #18, когда пользователь выставляет селектор выбора диапазона выше 500 - пороговое значение, которые я хочу, чтобы останавливало поток данных (Я точно не знаю зачем, это странный компонент для примера).

Безобразие здесь в том, что я императивно управляю отпиской в нескольких разных местах в этом довольно тривиальном примере.

Единственное реальное преимущество такого подхода была бы производительность. Так как вы использовали меньше абстракций чтобы реализовать код, он, вероятно, выполнится немного шустрее. Это врядли будет иметь заметный эффект в большинстве веб-прирложений, и я не думаю что это то, о чём стоит беспокоиться.

С другой стороны, вы можете всегда комбинировать подписки в одну единственную подписку с помощью создания родительской подписки и добавления к ней других дочерних. Но к концу дня, вы скорее всего будете делать всё ту же вещь, что и в начале, и вероятность запутаться очень велика. 

## Компонуйте управление подписками с помощью takeUntil

Сейчас давайте переделаем тот же простой пример, только с использованием RxJS оператора `takeUntil`:

```js
class MyGenericComponent extends SomeFrameworkComponent {
 updateData(data) {
  // здесь обновляем компонент способом, который зависит от используемого фреймворка
 }

 onMount() {
   const data$ = this.getData();
   const cancelBtn = this.element.querySelector(‘.cancel-button’);
   const rangeSelector = this.element.querySelector(‘.rangeSelector’);
   const cancel$ = Observable.fromEvent(cancelBtn, 'click');
   const range$ = Observable.fromEvent(rangeSelector, 'change').map(e => e.target.value);
   
   const stop$ = Observable.merge(cancel$, range$.filter(x => x > 500))
   this.subscription = data$.takeUntil(stop$).subscribe(data => this.updateData(data));
 }

 onUnmount() {
  this.subscription.unsubscribe();
 }
}
```

Первое что можно заметить - кода стало меньше. Это только одно преимущество. Другое - мы получили композицию потока `stop$` событий которые заканчивают поток данных. Это значит, что кода я решу добавить другое условие для закрытия потока данных, например, скажем, по таймеру, я просто сделаю слияние нового наблюдаемого объекта в `stop$`. Ещё одно преимущество довольно очевидно - у меня есть только один объект подписки, которым я управляю императивно [От этой единственной подписки мы избавиться не можем]. С этим ничего не поделать, потому что в этом месте функциональное программирование встречается с объектно-ориентированным миром. JavaScript императивный язык в конце концов, и мы должны встретиться с этим в какой-то момент.

Другое преимущество этого подхода - это фактическое завершение наблюдаемого объекта. Это означает что происходит событие завершения, которое может быть обработано в любое время, когда вы захотите завершить ваш наблюдаемый объект. Если вы просто вызываете `unsubscribe` на возвращённом объекте подписки, нет никакой возможности получить уведомление о том, что отписка произошла. Однако, если вы используете `takeUntil` (или другие операторы перечисленные ниже), вы будете уведомлены в обработчике завершения [аргумент `onCompleted` функции [`subscribe`](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/operators/subscribe.md)] что наблюдаемый объект был остановлен. 

Последнее преимущество - это то, что в действтительности вы связываете всё в одном месте одним вызовом `subscribe`. Это выгодно потому что становится намного-намного легче найти - где вы начали вашу подписку в коде. Помните, что наблюдаемые объекты не делают ничего пока вы не подпишитесь на них, и по-этому точка подписки - очень важное место в коде.

Есть один недостаток с точки зрения семантики RxJS, но об этом едва стоит беспокоиться перед лицом перечисленных преимуществ. Недостаток в семантике заключается в том, что завершение наблюдаемого объекта - это знак, что производитель событий хочет сказать потребителю, что он выполнен, а отписка означает, что потребитель сообщаяет производителю, что он больше не нуждается в данных.

Так же немного в худшую сторону будет очень небольшая разница в производительности между новым подходом и просто императивным вызовом `unsubscribe`.

## Другие операторы

Есть много других путей, чтобы закончить поток более естественным для Rx пособом. Я рекомендую познакомиться по меньшеу мере со следующими операторами: 

* take(n): производит N значений перед тем как остановить наблюдаемый объект.
* takeWhile(predicate): проверяет производимое значение с помощью функции-предиката, если она возвращет `false`, поток будет завершён.
* first(): производит первое значение и завершает поток.
* first(predicate): проверяет каждое значение с помощью функции-предиката, если какое-то из них вернёт `true`, значение будет произведено и поток завершён.

## Вывод: используйте `takeUntil`, `takeWhile`, и другие.

Вам следовало бы так же использовать операторы вроде `takeUntil` для управления вашими подписками RxJS. Как правило большого пальца - если вы видите две или больше подписки которыми собираетесь управлять в одном компоненте, вам следует задаваться вопросом - возможно ли сделать их комбинирование лучше:

* более легко компонуемыми;
* с событием завершения когда вы завершаете поток;
* в целом с меньшим количество кода;
* с меньшим явным управлением;
* с меньшим количеством фактических точек подписки (из-за меньшего количества вызовов `subscribe`).

Перевод основан на <a target="_blank" href="https://medium.com/@benlesh/rxjs-dont-unsubscribe-6753ed4fda87">RxJS: Don’t Unsubscribe</a> by <a target="_blank" href="https://medium.com/@benlesh">Ben Lesh</a>.