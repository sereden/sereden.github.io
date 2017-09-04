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