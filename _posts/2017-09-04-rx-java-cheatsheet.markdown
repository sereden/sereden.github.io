---
layout: post
title:  "RxJava Cheetsheet"
date:   2017-09-04 12:27:49 +0300
categories: rxjava
---
Cache
====================
`cache()` - не створює новий `Observable`, коли підписується новий subscriber, але пересилає йому всі попередні івенти.

{% highlight java %}
Observable<Integer> ints =
	 Observable.create(subscriber -> {
		 log("Create");
		 subscriber.onNext(42);
		 subscriber.onCompleted();
		 }
	 );
log("Starting");
ints.subscribe(i -> log("Element A: " + i));
ints.subscribe(i -> log("Element B: " + i));
log("Exit");
{% endhighlight %}

Результуючий лог:
```
main: Starting
main: Create
main: Element A: 42
main: Create
main: Element B: 42
main: Exit
```
Якщо добавити `cache()`, то буде (зверніть увагу, що 4 стрічка попереднього прикладу не визвалася):

```
main: Starting
main: Create
main: Element A: 42
main: Element B: 42
main: Exit
```

Infinite Streams
================
Такий код може породжувати Memory leaks
{% highlight java %}
// DON'T
Observable<BigInteger> naturalNumbers = Observable.create(
	 subscriber -> {
		 BigInteger i = ZERO;
		 while (true) { //don't do this!
			 subscriber.onNext(i);
			 i = i.add(ONE);
	 	}
 });
naturalNumbers.subscribe(x -> log(x));
{% endhighlight %}

Краще:
{% highlight java %}
// Code as previous
while (!subscriber.isUnsubscribed()) {
	 subscriber.onNext(i);
	 i = i.add(ONE);
 }
 {% endhighlight %}

rx.subjects.Subject
=================

`Subject extends Observable implements Observer` - одночасно і породжують і отримують дані
{% highlight java %}
private final PublishSubject<Status> subject = PublishSubject.create();
public TwitterSubject() {
	TwitterStream twitterStream = new TwitterStreamFactory().getInstance();
	twitterStream.addListener(new StatusListener() {
		@Override
		public void onStatus(Status status) {
			subject.onNext(status);
		}
		@Override
		public void onException(Exception ex) {
			subject.onError(ex);
		}
	});
	twitterStream.sample();
}

private void doSomething(){
	subject.filter(status -> status.successful)
		.subscribe(status -> log(status.getName()));
}
{% endhighlight %}

### AsyncSubject
**Запамятовує останній** породжений елемент та віддає його її підписникам, коли `onComplete()` визивається. Поки `AsyncSubject` не завершено всі елементи окрім останнього віхиляються
### BehaviorSubject
Працює так само як `PublishSubject` лише в різниці, що при підписці **отримує найостанніший елемент**, який був породжений перед цим. Це довзоляє Subscriber'у зразу знати стан. Наприклад, новий підписник наблюдає за зарядом пристрою. При підписці він отримає останнє значення.
### ReplaySubject
Такий як `BehaviorSubject`, але отримує всі івенти від початку існування.
*Небезпечний* оскільки може породити Memory leaks, якщо потік безкінечний.

*Розгляньте:*

* createWithSize()
* createWithTime()
* createWithTimeAndSize()

doOn(Next/Error/Subscribe)()
=====
Wiretap патерн, який просто наглядає над породженим обєктом без впливу на нього і передає далі. В теорії може змінювати стан обєкта, але це може мати *катастрофічний вплив*

publish().refCount() = share()
===========
В теорії має бути як і `cache()`, але не впевнений :-). Кажуть, що він не трогає оригінальний `Observable` і рахує скільки subscriber'ів є. Якщо змінюється з 0 на 1, то відбується підписка і дані починають надходити. А якщо підключиться, ще один subscriber, то нової ініціалізації не буде, новий буде отримувати ті ж дані, що і перший subscriber. Якщо останній subscriber відпишеться - Observer перестане породжувати дані. Виглядає ніби subscriber шейрять (share) один Observer, тому і зявився аліас - `share()` (який виглядає як `publish().refCount()`

map()
=======
Трансформує один тип даних в інший. Трансормація **блокується**, тому виконання має бути швидким
{% highlight java %}
Observable<Instant> instants = tweets
 	.map(Status::getCreatedAt)
 	.map((Date d) -> d.toInstant());
{% endhighlight %}

flatMap()
======
На відміну від `map()` вертає `Observable` (з яким також можна робити якісь операції)
Треба його використовувати, коли:

* Необхідні довгі, асинхронні **операції без блокування**
* Необхідна трансформація один-до-багатьох: кожний *Employee* трансформується в потік виконаних задач, коли 1 Employee може мати n виконаних задач

Також `flatMap()` не дає гарантії щодо збереження порядку трансформації вхідних данних. Якщо потрібно порядок даних важливий - див. `concatMap()`
{% highlight java %}
Observable
 .just(DayOfWeek.SUNDAY, DayOfWeek.MONDAY)
 .flatMap(this::loadRecordsFor);
 
 ///
 Observable<String> loadRecordsFor(DayOfWeek dow) {
	  switch(dow) {
		  case SUNDAY:
			  return Observable
			  .interval(90, MILLISECONDS)
			  .take(5)
			  .map(i -> "Sun-" + i);
		  case MONDAY:
			  return Observable
			  .interval(65, MILLISECONDS)
			  .take(5)
			  .map(i -> "Mon-" + i);
		  //...
	  }
 }
 {% endhighlight %}
Результат:
```
Mon-0, Sun-0, Mon-1, Sun-1, Mon-2, Mon-3, Sun-2, Mon-4, Sun-3, Sun-4
```

За замовчуванням, `flatMap()` не має обмеження на кількість даних, які повинні трансформуватися. 
`flatMap(User::loadProfile, 10)` зупиняє породжування 11го обєкту. Потрібно використовувакти, коли трансформуюча функція потребує **багато ресурсів**, які можуть привести то *OutOfMemoryException*.

concatMap()
=====
На відміну від `flatMap()` трансформує всі вхідні дані у заданому порядку і результат прикладу даного у `flatMap()` є такий:
```
Sun-0, Sun-1, Sun-2, Sun-3, Sun-4, Mon-0, Mon-1, Mon-2, Mon-3, Mon-4
```
Важливо розуміти, що `concatMap()` не вносить жодного паралелізму, але **зберігає природній порядок надходжння обєктів**

timer()
====
Засинає на вказаний період, перед тим як породжувати обєкти

interval() 
====
Генерує зрозстаючі числа від 0 з вказаною затримкою

delay()
=====
не породжує обєкти зразу після підписки, а **затримає на вказаний період**. 
Насправді обгортка над `timer()` + `flatMap()`
{% highlight java %}
Observable
	.timer(1, TimeUnit.SECONDS)
	.flatMap(i -> Observable.just(x, y, z))
{% endhighlight %}

Але на відміну від `timer()` **вираховує затримку для кожного івенту**, що приходить ніж для всього потоку.  
