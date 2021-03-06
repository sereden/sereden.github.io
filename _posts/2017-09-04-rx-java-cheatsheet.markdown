---
layout: post
title:  "RxJava Cheatsheet"
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
Працює так само як `PublishSubject` лише в різниці, що при підписці **отримує найостанніший елемент**, який був породжений перед цим. Це довзоляє Subscriber'у зразу знати стан. Наприклад, новий підписник наглядає за зарядом пристрою. При підписці він отримає останнє значення.
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

publish() and connect()
====
`publish()` Дозволяє не запускати потік, якщо хтось підписався, дані почнуться емітитися лише, коли визваний `connect()`
![publish-connect](http://reactivex.io/documentation/operators/images/publishConnect.c.png)

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
`flatMap(User::loadProfile, 10)` зупиняє породжування 11го обєкту. Потрібно використовувакти, коли трансформуюча функція потребує **багато ресурсів**, які можуть привести до *OutOfMemoryException*.

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
не породжує обєкти зразу після підписки, а **затримує на вказаний період**. 
Насправді обгортка над `timer()` + `flatMap()`
{% highlight java %}
Observable
	.timer(1, TimeUnit.SECONDS)
	.flatMap(i -> Observable.just(x, y, z))
{% endhighlight %}

Але на відміну від `timer()` **вираховує затримку для кожного обєкту**, що приходить ніж для всього потоку.  

merge(), .mergeWith()
=====
Обєднує два потоки **в один результуючий**. Очевидно, `merge()` переконується, що обєкти не накладаються, якщо були породжені одночасно.

**Помилка**, що зявилася в будь-якому Observable **передається далі** наглядачам.
`mergeDelayError()` **відкладе помилку** аж доки всі інші потоки не закінчаться. 
Більш того `mergeDelayError()` збере **всі** помилки і помістить їх в `CompositeException`

zip(), .zipWith()
====
Приймає два і більше потоків і комбінує їх один з одним таким способом, що **кожний елемент з першого потоку поєднується з відповідним елементом другого потоку**

{% highlight java %}
Observable<Temperature> temperatureMeasurements = station.temperature();
Observable<Wind> windMeasurements = station.wind();
temperatureMeasurements
 .zipWith(windMeasurements,
 	(temperature, wind) -> new Weather(temperature, wind));
 {% endhighlight %}
 
Якщо кожний Observable, що приймає `zip()` має велику **різницю у швидкості** обробки даних і кількість вхідних даних також велика, може бути проблема в накопичувані великої кількості інформації, що може привести до **Memory Leaks**
 
combineLatest()
====
Бере останій елемент з кожного потоку і комбінує в результуючий
![Combine latest](http://reactivex.io/documentation/operators/images/combineLatest.png)
{% highlight java %}
Observable<String> fast = interval(10, MILLISECONDS).map(x -> "F" + x);
Observable<String> slow = interval(17, MILLISECONDS).map(x -> "S" + x);
Observable.combineLatest(
 	slow,
 	fast,
 	(s, f) -> f + ":" + s
).forEach(System.out::println);
 {% endhighlight %}
 
 ```
 F0:S0
 F1:S0
 F2:S0
 F2:S1
 F3:S1
 F4:S1
 F4:S2
 F5:S2
 F5:S3
 ....
 ```
Зверніть увагу, що **деякі породжені обєкти упускаються**, якщо вони не були останні.
 
Також `combineLatest()` все одно з якого потоку прийшли дані, на відміну від `withLatestFrom()`
 
withLatestFrom()
====
Вибирає **пріоритетний потік з якого завжди будуть братися обєкти** і які не будуть упускатися. Приклад із `combineLatest()`, але замінений на `withLatestFrom()`:
{% highlight java %}
slow
 .withLatestFrom(fast, (s, f) -> s + ":" + f)
{% endhighlight %}
```
S0:F1
S1:F2
S2:F4
S3:F5
S4:F7
S5:F9
S6:F11
```

startWith()
====
вставляє певне константне значення перед оригінальним обєктом

{% highlight java %}
Observable<String> fast = interval(10, MILLISECONDS)
 	.map(x -> "F" + x)
 	.delay(100, MILLISECONDS)
 	.startWith("FX");
Observable<String> slow = interval(17, MILLISECONDS).map(x -> "S" + x);
slow
 	.withLatestFrom(fast, (s, f) -> s + ":" + f)
 	.forEach(System.out::println);
 {% endhighlight %}
 
 ```
 S0:FX
 S1:FX
 S2:FX
 S3:FX
 S4:FX
 S5:FX
 S6:F1
 S7:F3
 S8:F4
 S9:F6
 ```
 
amb(), .ambWith()
====
Жде потік, який перший породить обєкт і **починає слухати лише його, відхиляючи всі інші.**
![amb](http://reactivex.io/documentation/operators/images/amb.png)
Верхній потік **перший породив обєкт**, тому від нижнього автоматично **іде `unsubscribe`**

scan()
====
**Агрегує (сумує) всі попередньо породжені дані** і вертає їх разом із наступним породженим обєктом.
![scan](http://reactivex.io/documentation/operators/images/scan.png)

{% highlight java %}
Observable<Long> progress = // [10, 14, 12, 13, 14, 16]
Observable<Long> totalProgress = /* [10, 24, 36, 49, 63, 79]
 10
 10+14= 24
 	24+12=36
 		36+13=49
 	  		49+14=63
 				63+16=79
*/

Observable<Long> totalProgress = progress
	.scan((total, chunk) -> total + chunk);
{% endhighlight %}

Також можна задавати **початкове значення**
{% highlight java %}
Observable<BigInteger> factorials = Observable
 .range(2, 100)
 .scan(BigInteger.ONE, (big, cur) ->
 big.multiply(BigInteger.valueOf(cur)));
 {% endhighlight %}
 
Якщо неважливі проміжні значення - `reduce()`
 
reduce()
====
**Агрегує (сумує) дані** так само як і `scan()` але в **результаті емітить лише кінцевий результат (суму)**.
Будьте уважні, якщо потік безкінечний, то ніякі дані не прийдуть.
`reduce` можна предаставити так:
{% highlight java %}
public <R> Observable<R> reduce(R initialValue,  Func2<R, T, R> accumulator) {
 return scan(initialValue, accumulator).takeLast(1);
}
{% endhighlight %}
Якщо потрібно мати справу із mutable даними - див `collect()`

collect()
====
те ж саме, що й `reduce()` але в якості аккумулятора використовується **`mutable` структура**.
Наприклад, це виглядає некрасиво:
{% highlight java %}
Observable<List<Integer>> all = Observable
 .range(10, 20)
 .reduce(new ArrayList<>(), (list, item) -> {
 	list.add(item);
 return list;
 });
 {% endhighlight %}
 Але замінити на `return list.add(item)` не можна оскільки поверється не `ArrayList`, а `boolean`. Тому для краси і придумали `collect()`:
 {% highlight java %}
 Observable<List<Integer>> all = Observable
  .range(10, 20)
  .collect(ArrayList::new, List::add);
{% endhighlight %}

Ще простіше - `toList()`

Ще один приклад:

{% highlight java %}
Observable<String> str = Observable
 .range(1, 10)
 .collect(
 	StringBuilder::new,
 	(sb, x) -> sb.append(x).append(", "))
 .map(StringBuilder::toString);
 {% endhighlight %}
 
single()
====
Не породжує жодних даних, лише перевіряє чи точно було **створено лише один обєкт**. Якщо це не так, то буде породжено виключення.

distinct()
====
Автоматично **відхиляє обєкти, які попередньо вже були породжені**. Порівнюються обєкти завдяки `equals() i hashCode()`
Тримає в памяті весь ланцюг данних, тому може визвати `OutOfMemoryException`

distinctUntilChanged()
====
Обєкт відхиляється, якщо **попередній був точно такий же** (`equals()`)

Приклад, що емітить обєкти лише тоді, коли погода змінилася

{% highlight java %}
Observable<Weather> measurements = //...
Observable<Weather> tempChanges = measurements
 .distinctUntilChanged(Weather::getTemperature);
 {% endhighlight %}
 На відміну від `distinct()` тримає в памяті останній елемент для порівняння.
 
take(n) і skip(n)
====
Бере перших n елементів / пропускає перші n елементів відповідно

takeLast(n) і skipLast(n)
====
Перший бере `n`тий обєкт, тому всі попередні обєкти мають зберігатися в памяті.

Другий в свою чергу пропускає `n` обєктів 

concat(), .concatWith()
=====
Дозволяє обєднувати два потоки: **коли перший закінчився, `concat()` підписується до другого**

**Важливо:** `concat()` підпишеться на другий потік лише тоді, коли перший закінчився.
![concat](http://reactivex.io/documentation/operators/images/concat.png)

{% highlight java %}
Observable<Car> fromCache = loadFromCache();
Observable<Car> fromDb = loadFromDb();
Observable<Car> found = Observable
 .concat(fromCache, fromDb)
 .first();
 {% endhighlight %}
 У даному прикладі, якщо fromCache() верне хоча б 1 елемент **підписка на `fromDb` не відбудеться**

switchOnNext()
=====
має сенс лише коли ми працюємо з вхідним обєктом типу `Observable<T>`. Коли наступний потік `Observable<T>` зявляється, `switchOnNext()` відписується від попереднього і підписується до нового.  
{% highlight java %}
//A
map(innerObs ->
 innerObs.delay(rnd.nextInt(5), SECONDS))
//B
flatMap(innerObs -> just(innerObs)
 .delay(rnd.nextInt(5), SECONDS))
 {% endhighlight %}
 В прикладі `A` ми лише затримуємо породження наступуного обєкта `innerObs`, натомість в `B` ми затримуємо породження нового `Observable<T>`. Логічно, що при реалізації першого варіанту `switchOnNext()` нічого не зробить, оскільки має змінитися саме `Observable` який емітить дані.
![switchOnNext](http://reactivex.io/documentation/operators/images/switchDo.png)

compose()
====
Бере функцію в якості аргумента, яка повина трасформувати вхідний потік через серію інших операторів. Грубо кажучи **реалізація кастомного оператора, на основі використання існуючих операторів**
{% highlight java %}
private <T> Observable.Transformer<T, T> odd() {
 Observable<Boolean> trueFalse = just(true, false).repeat();
 return upstream -> upstream
 .zipWith(trueFalse, Pair::of)
 .filter(Pair::getRight)
 .map(Pair::getLeft);
}
//...
//[A, B, C, D, E...]
Observable<Character> alphabet =
 Observable
 .range(0, 'Z' - 'A' + 1)
 .map(c -> (char) ('A' + c));
//[A, C, E, G, I...]
alphabet
 .compose(odd())
 .forEach(System.out::println);
 {% endhighlight %}

lift()
====
Майже як і `compose()`, але дозволяє будувати свої правила трансформації потоку, але управління повністю належить нашому кастомному оператору.

BlockingObservable
====
Потрібний тоді, коли у нас немає можливості переписати весь існуючий код на rx. Ключовий опертор - `toBlocking()` який визивається в блокуючому потоці. Грубо кажучи визиває весь ланцюг rx методів в головному потоці. 
{% highlight java %}
List<Person> people = personDao
	 .listPeople()
	 .toList()
	 .toBlocking()
	 .single();
{% endhighlight %}

Старайтеся **ніколи** не використовувати

onErrorResumeNext()
====
оператор, що відловлює помилки, що виникають у попередніх потоках і **надає запасний `Observable`**

{% highlight java %}
void bestBookFor(Person person) {
	 recommend(person)
		 .onErrorResumeNext(bestSeller())
		 .map(Book::getTitle)
		 .subscribe(this::display);
}
{% endhighlight %}

![onErrorResumeNext](http://reactivex.io/documentation/operators/images/onErrorResumeNext.js.png)

Schedulers
====
### Schedulers.newThread()
найгріший варіант використання, оскільки **створені нові потоки ніяк не контролюється**. Єдиний приклад використання у систем з велико-зернистою моделлю: займає багато часу на виконання кожного потоку, але їх мало, тому ймовірніть перевикористання мала.
### Schedulers.io()
Схожий до `newThread()`, але **запущені потоки можуть перевикористовуватися**, що позитивно впоиває на швидкодію. Потік має низький пріоритет, тому він підходить для виконання не CPU-повязаних операцій таких як **зчитування з інтернету, диску, засипання**

### Schedulers.computation()
Використовується для **CPU-повязаних операцій**, коли потрібно виконати великі обрахунки не блокуючого коду (не зчитувати з диску, інтернету і тд) 
Є можливість зменшити максимальну к-сть потоків, використовуючи rx.scheduler.max-computation-threads. К-сть потоків не може бути більша за кількість ядер

### Schedulers.from(Executor executor)
можливість кастомної реалізації пулу потоків

### Schedulers.immediate()
**Виконує таск в клієнтському потоці**. Використовується тоді, коли немає необхідності переключати потоки, але дизайн вимагає певного `Scheduler'a`. Можна використовувати в тестах :-)
Також ставить всі породжені обєкти в чергу на виконання.

### Schedulers.trampoline()
Такий же як `immediate()`, але різниця в тому, що **`trampoline()` бере таск на виконання лише після того, як попередній був закінчений.**
`Trampoline` - це паттерн в функціональному програмуванні, що довзоляє реалізовувати **рекурсію без безкінечного росту стеку викликів**

Різниця між `immediate()` i `trampoline()`
{% highlight java %}
Scheduler scheduler = Schedulers.immediate();
Scheduler.Worker worker = scheduler.createWorker();

log("Main start");
worker.schedule(() -> {
	 log(" Outer start");
	 sleepOneSecond();
	 worker.schedule(() -> {
		 log(" Middle start");
		 sleepOneSecond();
		 worker.schedule(() -> {
			 log(" Inner start");
			 sleepOneSecond();
			 log(" Inner end");
		 });
		 log(" Middle end");
	 });
	 log(" Outer end");
 });
log("Main end");
{% endhighlight %}
`Worker` для `immediate()` виведе так:
 ```
 1029 | main | Main start
 1091 | main | Outer start
 2093 | main | Middle start
 3095 | main | Inner start
 4096 | main | Inner end
 4099 | main | Middle end
 4099 | main | Outer end
 4099 | main | Main end
 ```
 `Worker` для `trampoline()` виведе так, оскільки `Middle` не почнеться, поки попередній `Outer` не закінчиться і `Inner` не буде виконувати допоки `Middle` не закінчиться
 ```
 1041 | main | Main start
 1095 | main | Outer start
 2099 | main | Outer end
 2099 | main | Middle start
 3101 | main | Middle end
 3101 | main | Inner start
 4102 | main | Inner end
 4102 | main | Main end
 ```
### Schedulers.test()
Використовується в тестах і ніколи не повинен бути в продактшині, дозволяє симулювати плин часу

subscribeOn()
====
- Декілька підряд визваних `subscribeOn()` не змінюють потік після першого, наприклад 
{% highlight java %}
log("Starting");
Observable<String> obs = simple();
log("Created");
obs
	 .subscribeOn(schedulerA)
	 //many other operators
	 .subscribeOn(schedulerB)
	 .subscribe(
		 x -> log("Got " + x),
		 Throwable::printStackTrace,
		 () -> log("Completed")
	 );
log("Exiting");
{% endhighlight %}
```
17 | main | Starting
73 | main | Created
83 | main | Exiting
84 | Sched-A-0 | Subscribed
84 | Sched-A-0 | Got A
84 | Sched-A-0 | Got B
84 | Sched-A-0 | Completed
```
Але `schedulerB` не ігнорується повністю. В ньому створюється `schedulerA` в якому і виконується вся робота. Тому **декілька `subscribeOn` - це лишній overhead** (толку 0, але ресурси затратилися).
- `subscribeOn()` - це не true <s>detective</s> concurrency. Воно **не гарантує паралельного виконання**, якщо було не використане правильно. 
  Наприклад, це буде виконане **в іншом потоці**, але **послідовно**:
 {% highlight java %}
 Observable
	  .just("bread", "butter", "milk", "tomato", "cheese")
	  .subscribeOn(schedulerA)
	  // flatMap не приносить паралелізм сам по собі
	  .flatMap(prod -> rxGroceries.purchase(prod, 1))
	  .reduce(BigDecimal::add)
	  .single();
 {% endhighlight %}
 
 ```
 144  | Sched-A-0 | Purchasing 1 bread
 1144 | Sched-A-0 | Done 1 bread
 1146 | Sched-A-0 | Purchasing 1 butter
 2146 | Sched-A-0 | Done 1 butter
 2146 | Sched-A-0 | Purchasing 1 milk
 3147 | Sched-A-0 | Done 1 milk
 3147 | Sched-A-0 | Purchasing 1 tomato
 4147 | Sched-A-0 | Done 1 tomato
 ```
 
 А таке використання дозволить досягнути **паралелізму** як із `ForkJoinPool`
 {% highlight java %}
Observable<BigDecimal> totalPrice = Observable
	  .just("bread", "butter", "milk", "tomato", "cheese")
	  .flatMap(prod ->
		  rxGroceries
			  .purchase(prod, 1)
			  .subscribeOn(schedulerA))
	  .reduce(BigDecimal::add)
	  .single();
{% endhighlight %}

```
113  | Sched-A-1 | Purchasing 1 butter
114  | Sched-A-0 | Purchasing 1 bread
125  | Sched-A-2 | Purchasing 1 milk
125  | Sched-A-3 | Purchasing 1 tomato
126  | Sched-A-4 | Purchasing 1 cheese
1126 | Sched-A-2 | Done 1 milk
1126 | Sched-A-0 | Done 1 bread
1126 | Sched-A-1 | Done 1 butter
1128 | Sched-A-3 | Done 1 tomato
1128 | Sched-A-4 | Done 1 cheese
```

observeOn() vs subscribeOn()
====
`subscribeOn()` і `observeOn()` працюють дуже добре разом, коли ви **хочете фізично
відділити виробника (`Observable.create()`) та споживача (`Subscriber`)**.

Що це означає? `subscribeOn()` **змінює потік всіх верх стоячих операторів** до появи або `subscribeOn()` або `observeOn()`. `observeOn()` в свою чергу **змінює потік всіх нижче стоячих операторів** до появи наступного `observeOn()`.
{% highlight java %}
log("Starting");
final Observable<String> obs = simple();
log("Created");
obs
	 .doOnNext(x -> log("Found 1: " + x))
	 .observeOn(schedulerB)
	 .doOnNext(x -> log("Found 2: " + x))
	 .observeOn(schedulerC)
	 .doOnNext(x -> log("Found 3: " + x))
	 .subscribeOn(schedulerA)
	 .subscribe(
		 x -> log("Got 1: " + x),
		 Throwable::printStackTrace,
		 () -> log("Completed")
	 );
log("Exiting");
{% endhighlight %}
```
21  | main | Starting
98  | main | Created
108 | main | Exiting
129 | Sched-A-0 | Subscribed
129 | Sched-A-0 | Found 1: A
129 | Sched-A-0 | Found 1: B
130 | Sched-B-0 | Found 2: A
130 | Sched-B-0 | Found 2: B
130 | Sched-C-0 | Found 3: A
130 | Sched-C-0 | Got: A
130 | Sched-C-0 | Found 3: B
130 | Sched-C-0 | Got: B
130 | Sched-C-0 | Completed
```
так як `simple()` не має (хоча по-доброму мав би) `subscribeOn()` оператора, то `schedulerA` застосовується до всіх операторів зверху аж до появи `.observeOn(schedulerB)`

Інші способи використання Schedulers
====
є деякі оператори (interval(), range(), timer(), repeat(), skip(), take(), timeout(), delay()) які по **своїй природі блокують потік**. Логічно, якщо у нас є, наприклад, `delay()`, який породжує обєкти з заданою затримкою в `computation()` потоці, то даний потік буде блокований на вказаний період. Для уникнення цього потрібно використовувати інший потік:
{% highlight java %}
Observable
	 .just('A', 'B')
	 .delay(1, SECONDS, schedulerA)
	 .subscribe(this::log);
{% endhighlight %}

BackPressure
====

sample() (aka throttleLast()), throttleFirst())
=====
Забирає останній елемент, якщо такий зявився через заданий проміжок. Якщо протягом затримки було декілька елементів - оброблюється лише останній

{% highlight java %}
 Observable<String> names = Observable
        .just("Mary", "Patricia", "Linda",
            "Barbara",
            "Elizabeth", "Jennifer", "Maria", "Susan",
            "Margaret", "Dorothy");
    Observable<Long> absoluteDelayMillis = Observable
        .just(0.1, 0.6, 0.9,
            1.1,
            3.3, 3.4, 3.5, 3.6,
            4.4, 4.8)
.map(d -> (long)(d * 1_000)); //...
    delayedNames
        .throttleFirst(1, SECONDS)
        .subscribe(System.out::println);
{% endhighlight %}

```
Mary
Barbara
Elizabeth
Margaret
```

buffer()
====
обєднує декілька івентів в один `List` заданого розміру.
На відміну від `toList()` - породжує **декілька листів, замість одного**

{% highlight java %}
Observable
	.range(1, 7) //1, 2, 3 . . . 7 
	.buffer(3) 
	.subscribe((List<Integer> list) -> {
                System.out.println(list);
            }
);

{% endhighlight %}

```
[1, 2, 3]
[4, 5, 6]
[7]
```

**buffer(count, drop)** дозволяє сконфігурувати скільки **старих значень викинути** перед тим як передати далі:
{% highlight java %}
Observable
            .range(1, 7)
            .buffer(3, 1)
            .subscribe(System.out::println);
{% endhighlight %}

```
[1, 2, 3]
[2, 3, 4]
[3, 4, 5]
[4, 5, 6]
[5, 6, 7]
[6, 7]
[7]
```

window()
====
Майже те саме, як і `buffer()` тільки краще, тому його радять використовувати:
- Отримуючи int, групує івенти в списки фіксованого розміру
- Отримуючи `time unit`, групує події протягом періоду фіксованого часу
- Отримуючи `Observable`, позначає початок і кінець кожної пачки

`window()` на відміну від `buffer()` **групує обєкти не в список, а в `Observable`**

`buffer()` і `window()` об'єднують кілька подій разом. `sample()` вибирає одну досить довільну подію один раз за деякий час. Ці оператори не враховують, скільки часу минуло між подіями. Тому деякі **обєкти можуть бути відхилені**.

debounce() (alias: throttleWithTimeout())
====
Відхиляє івенти що слідують один за одним: тобто пропускає лише ті, що витримали вказаний період.
Наприклад, такий код не видасть ніякого результату, оскільки таймер `debounce()` не витримується:

{% highlight java %}
Observable
        .interval(99, MILLISECONDS)
        .debounce(100, MILLISECONDS)
{% endhighlight %}

onBackpressureBuffer()
====
Довзоляє **буферизувати 128 (за замовчуванням)** елементів, що емітяться. Але таке рішення може привести до `java.lang.OutOfMemoryError`.

Рекомендовано використовувати дану конструкцію, оскільки вона більш захищена `.onBackpressureBuffer(1000, () -> log.warn("Buffer full"))`. Але все одно воно не спасе, оскільки `MissingBackpressureException` буде викинуте після попередження.

onBackpressureDrop()
====
Просто **відхиляє всі івенти, які появилися без виклику `request()`** (тобто не були "спожиті" слухачем)
```
.onBackpressureDrop(dish -> log.warn("Throw away {}", dish))
````
 
onBackpressureLatest()
====
Такий самий як і `onBackpressureDrop()`, але **зберігає останній елемент, що був відхилений** тим самим дозволяє спасти найостанніший елемент у випадку пізнього `request()`

SyncOnSubscribe.createStateful and SyncOnSubscribe.createStateless
====
Довзоляють будувати **`backpressure` безпечні `Observable`**. Різниця лише параметрі, що позначає **стан попереднього елемента**
{% highlight java %}
Observable.OnSubscribe<Double> onSubscribe =
        SyncOnSubscribe.createStateless(
            observer -> observer.onNext(Math.random())
        );
    Observable<Double> rand = Observable.create(onSubscribe);
    
    //
    
    Observable.OnSubscribe<Long> onSubscribe =
                SyncOnSubscribe.createStateful(
                        () -> 0L,
                        (cur, observer) -> {
                            observer.onNext(cur);
    return cur + 1; }
                );
        Observable<Long> naturals = Observable.create(onSubscribe);
{% endhighlight %}

SyncOnSubscribe.createSingleState
====

{% highlight java %}
Observable.OnSubscribe<Object[]> onSubscribe = SyncOnSubscribe.createSingleState(
	() -> resultSet,
	(rs, observer) -> { 
		try {
			rs.next();
			observer.onNext(toArray(rs)); } 
		catch (SQLException e) {
			observer.onError(e);
		}
	}, 
	rs->{
		try {
			//Also close Statement, Connection, etc. 
			rs.close();
		} catch (SQLException e) { 
			log.warn("Unable to close", e);
		} 
	});
{% endhighlight %}


**Уникаючи затратних операцій в `subscribe()` ми зменшуємо потребу в `backpressure()`, але все одно краще подумати наперед.**

Погано:
{% highlight java %}
source.subscribe(this::store);
{% endhighlight %}
Так краще
{% highlight java %}
source
.flatMap(this::store)
.subscribe(uuid -> log.debug("Stored: {}", uuid));
{% endhighlight %}
Або
{% highlight java %}
source
	.flatMap(this::store) 
	.buffer(100) 
	.subscribe(hundredUuids -> log.debug("Stored: {}", hundredUuids))
{% endhighlight %}



{% highlight java %}
{% endhighlight %}