![ReactiveX logo](http://www.pvsm.ru/images/2015/12/19/realizaciya-MVVM-v-iOS-s-pomoshyu-RxSwift.jpg "ReactiveX")

Доброго времени суток, хабравчане. В этом цикле статей, я бы хотел рассказать о реактивном программировании, а именно о фреймворке
[RxSwift](https://github.com/ReactiveX/RxSwift) . На хабре и в сети были статьи по RxSwift, но на мой взгляд они слишком трудны для начинающих. Поэтому, если вы начинаете постигать реативное программирование в iOS, то прошу под кат.

<cut/>

Начнем с определения, что такое реактивное программирование. 
>Реактивное программирование — парадигма программирования, ориентированная на потоки данных и распространение изменений.

Так гласит нам великая [википедия](https://ru.wikipedia.org/wiki/Реактивное_программирование).

 Иными словами, в случае, когда мы программируем в императивном стиле, мы пишем в коде набор команд, которые должны выполняться последовательно. Реактивный стиль программирования придерживается несколько иных концепций. При реактивном стиле программирования наша программа является "слушателем" изменений состояний у наших наблюдаемых объектов. Звучит сложновато, но это не так, достаточно просто проникнуться этой концепции и все станет крайне легко и понятно, ~~пока нет багов~~.

 Я не буду расписывать как установить фреймворк, это легко сделать перейдя по [ссылке](https://github.com/ReactiveX/RxSwift). Давайте приступим к практике.
 
## Observable

 Начнем с простого,  но важного, наблюдаемый объект или Observable. Observable это то, что будет отдавать нам данные, нужен он для генерации  потока данных. 
```swift
let observable = Observable<String>.just("Первый observable")
```
**BINGO**! Мы создали первый observable. 

![и что?](http://risovach.ru/upload/2013/12/mem/moe-lico_38125824_orig_.jpeg)

Так как мы создали наблюдаемый объект, то логично,  что нам необходимо создать объект, который будет наблюдать.

```swift
let observable = Observable<String>.just("Первый observable")

_ = observable.subscribe { (event) in
    print(event)
}
```
в лог мы получаем следующее:  
```
next(Первый observable)  
completed
```
![completed?](https://habrastorage.org/webt/6d/8y/mo/6d8ymogkm6b0yvgdmvryssbv4ha.jpeg)

Observable отправляет нам информацию о своих event'ах, есть всего 3 вида:
- next
- error
- completed

Вместе с _next_'ом приходит элемент, который мы отправляли  и все события посланные нами, _error_ посылается как понятно из названия в случае ошибки, а _completed_ в случае, когда наш observable отослал все данные и завершает работу.  
Мы можем создать более детального ~~наблюдателя~~ subscriber'а и получить более удобный вид для обработки всех событий.

```swift
_ = observable.subscribe(onNext: { (event) in
    print(event)
}, onError: { (error) in
    print(error)
}, onCompleted: {
    print("finish")
}) {
    print("disposed") 
    //о том, что это такое и зачем это мы поговорим позже
}
```
```
Первая последовательность
finish  
disposed
```
В observable можно создавать последовательность не только из одной строки, да и вообще не только из строк, мы можем положить туда любой тип данных.

```swift
let sequence = Observable<Int>.of(1, 2, 4, 5, 6)
        
_ = sequence.subscribe { (event) in
    print(event)
}
```
```
next(1)
next(2)
... 
completed
```

Observable можно создать из массива значений.

```swift
let array = [1, 2, 3]
        
let observable = Observable<Int>.from(array)
      
_ = observable.subscribe { (event) in
    print(event)
}
```
```
next(1)
next(2)  
next(3)
completed
```

У одного _observable_ может быть скольугодно много _subscriber'ов_. А теперь терминология, что такое Observable?

**Observable** - это основа всего Rx, которая асинхронно генерирует последовательность неизменяемых данных и позволяет подписываться на нее другим.

## Disposing
Теперь, когда мы умеем создавать последовательность и подписываться на них, необходимо разобраться с такой штукой, как _disposing_.

Важно помнить, что _Observable_ это "холодный" тип, то есть наш observable не "испускает" никаких событий, пока на него не подпишутся. Observable существует до тех пор, пока он не пошлет сообщение об ошибке (_error_) или сообщение о завершении (_completed_). Если мы хотим явно отменить подписку, то можем следать следующее.  
```swift
//вариант №1

//создали массив значний
let array = [1, 2, 3]

//создали observable из массива значений
let observable = Observable<Int>.from(array)

//создали подписку на observable
let subscription = observable.subscribe { (event) in
    print(event)
}

//dispos'им нашу одноразовую подписку
subscription.dispose()
```

Есть более ~~красивый~~ правильный вариант.

```swift
//создаем сумку "утилизации"
let bag = DisposeBag()
//создали массив значний
let array = [1, 2, 3]

//создали observable из массива значений
let observable = Observable<Int>.from(array)

//создали подписку на observable
_ = observable.subscribe { (event) in
    print(event)
}.disposed(by: bag)
```

Таким образом мы добавляем нашу подписку в сумку утилизации или в _DisposeBag_.   
Для чего это нужно? Если вы используя подписку не добавите ее в сумку или явно не вызовите _dispose_, ну или в крайнем случае не приведете каким-то образом _observable_ к завершению, то скорее всего вы получите утечку памяти. DisposeBag вы будете использовать очень часто в своей работе с RxSwift.

## Operators

В функционально-реактивном программировании (ФРП далее) есть много встроенных операторов для трансформации элементов observable. Существует сайт [rxmarbles](http://rxmarbles.com), на нем можно посмотреть работу и эффект всех операторов, ну а мы все же рассмотрим некоторые из них.

### Map
Оператор [_**map**_](http://rxmarbles.com/#map) используется очень часто и думаю, что знаком многим, с его помощью мы трансформируем все полученные элементы.  
_Пример:_
```swift
let bag = DisposeBag()

let array = [1, 2, 3]
        
let observable = Observable<Int>.from(array).map { $0 * 2 }
        
_ = observable.subscribe { (e) in
    print(e)
}.disponsed(by: bag)
```
Что получим в консоли:
```
next(2)
next(4)
next(6)  
completed
```
Мы берем каждый элемент последовательности и создаем новую результирующую последовательность. Чтобы было более понятно, что происходит лучше записать подробнее.

```swift
let bag = DisposeBag()

let observable = Observable<Int>.from(array)

//создаем новый observable
let transformObservable = observable.map { $0 * 2 }
        
_ = transformObservable.subscribe { (e) in
    print(e)
}.disposed(by: bag)
```

#### Что такое "$0"?  
 $0 это название элемента поумолчанию, мы можем использовать в методах сокращенную и полную запись, чаще всего используется сокращенная запись.
 
```swift
 
 //сокращенная форма
 let transformObservable = observable.map { $0 * 2 }
 
 //полная форма
 let transformObservable = observable.map { (element) -> Int in
    return element * 2
}
```
 
 Согласитесь, что сокращенная форма записи куда проще, так?
 
### Filter
 Оператор [filter](http://rxmarbles.com/#filter) позволяет нам отфильтровать испускаемые нашим observable'ом данные, то есть при подписке мы не будем получать ненужные нам значения.  
**Пример:**
```swift
let bag = DisposeBag()

let array = [1, 2, 3, 4, 5 , 6, 7]
//создаем observable из массива
let observable = Observable<Int>.from(array)
//применяем функцию filter, сохраняя резултат в новый observable
let filteredObservable = observable.filter { $0 > 2 }
//подписка
_ = filteredObservable.subscribe { (e) in
    print(e)
}.disposed(by: bag)
```
Что мы получим в консоль?
```
next(3)
next(4)  
next(5)
... 
completed
```


Как мы видим, в консоли у нас только те значения, что удовлетворяют нашим условиям.

Кстати, операторы можно комбинировать, вот как это выглядело бы, если бы мы захотели применить сразу и оператор фильтрации и оператор _map_.
```swift
let bag = DisposeBag()

let array = [1, 2, 3, 4, 5 , 6, 7]
        
let observable = Observable<Int>.from(array)
        
let filteredAndMappedObservable = observable
    .filter { $0 > 2 }
    .map { $0 * 2 }
        
        
_ = filteredAndMappedObservable.subscribe { (e) in
    print(e)
}.disposed(by: bag)
```
**Консоль:**
```
next(6)
next(8)
next(10)
next(12)
next(14)
completed
```

### Distinct
Еще один отличный оператор, который связан с фильтрацией, оператор [disctinct](http://rxmarbles.com/#distinct) позволяет пропускать только измененные данные, лучше всего сразу обратиться к примеру и все станет понятно.

```swift
let bag = DisposeBag()

let array = [1, 1, 1, 2, 3, 3, 5, 5, 6]
        
let observable = Observable<Int>.from(array)
        
let filteredObservable = observable.distinctUntilChanged()        
        
_ = filteredObservable.subscribe { (e) in
    print(e)
}.disposed(by: bag)
```
В консоль мы получим следующее:
```
next(1)
next(2)
next(3)
next(5)
next(6)
completed
```
то есть в случае, если нынещний элемент последовательности идентичен предыдущему, то он пропускается и так происходит до тех пор, пока не появится отличный от предыдущего элемент, это очень удобно при работах скажем с UI, а именно с таблицей, в случае если нам пришли данные, такие же, как мы имеем сейчас, то _reload'ить_ таблицу не следует.

### TakeLast
Очень простой оператор [takeLast](http://rxmarbles.com/#takeLast), мы берем n-ое количество элементов с конца.

```swift
let bag = DisposeBag()

let array = [1, 1, 1, 2, 3, 3, 5, 5, 6]
        
let observable = Observable<Int>.from(array)
        
let takeLastObservable = observable.takeLast(1)        
        
_ = takeLastObservable.subscribe { (e) in
    print(e)
}.disposed(by: bag)
```

В консоль получим следующее:
```
next(6)
completed
```

### Throttle и Interval
Тут я решил взять сразу 2 оператора, просто потому, что с помощью второго, легко можно показать работу первого.

Оператор [throttle](http://rxmarbles.com/#throttle) позволяет взять паузу между захватом передаваемых значений, очень просто пример, вы пишите реактивное приложение, используете строку поиска и не хотите каждый раз после ввода данных пользователем либо перезагружать таблицу, либо лезть на сервер, поэтому вы используете _throttle_ и таким образом говорите, что хотите брать данные пользователя раз в 2 секунды (пример, можно поставить любой интервал) и снижаете расход ресурсов на лишнюю обработку, как это работает и описывается в коде? Смотрите ниже пример.

```swift
let bag = DisposeBag()
//observable будет генерировать значение каждые 0.5 секунды с шагом 1 начиная от 0
let observable = Observable<Int>.interval(0.5, scheduler: MainScheduler.instance)
        
let throttleObservable = observable.throttle(1.0, scheduler: MainScheduler.instance)
        
        
_ = takeLastObservable.subscribe { (event) in
    print("throttle \(event)")
}.disposed(by: bag)
```

В консоли будет:
```
throttle next(0)
throttle next(2)
throttle next(4)
throttle next(6)
...
```
Оператор [interval](http://rxmarbles.com/#interval) заставялет observable генерировать значения каждые 0,5 секунды с шагом 1 начиная с 0, вот такой простой таймер у Rx. Получается раз значения генерируются каждые 0,5 секунды, то в секунду генерируется 2 значения, нехитрая арифметика, а оператор throttle ждет секунду и берет последнее значение.

### Debounce
[Debounce](http://rxmarbles.com/#debounce) очень похож на предыдущий оператор, но чуть более умнее, на мой взгляд. Оператор debounce ждет n-ое количество времени и в случае, если со старта его таймера не было изменений, то берет последнее значение, если же мы пошлем значение, то таймера перезапустится снова. Это как раз очень полезно для ситуации описанной в предыдущем примере, пользователь вводит данные, мы ждем когда он закончит (если пользователь бездействует секунду или полторы), а потом начинаем выполнять какие-то действия. Поэтому если мы просто поменяем оператор в предыдущем коде, то значений мы не получим в консоль, потому что debounce будет ждать секунду, но каждые 0,5 секунды будет получать новое значение и перезапускать свой таймер, таким образом мы ничего не получим. Посмотрим пример.

```swift
let bag = DisposeBag()

let observable = Observable<Int>.interval(1.5, scheduler: MainScheduler.instance)
        
let debounceObservable = observable.debounce(1.0, scheduler: MainScheduler.instance)
    
_ = debounceObservable.subscribe({ (e) in
    print("debounce \(e)")
}).disposed(by: bag)
```
На этом этапе предлагаю закончить с операторами, в фреймворке RxSwift их очень много, нельзя сказать, что все из них очень нужны в повседнейной жизни, но знать о их существовании все же надо, поэтому желательно ознакомиться с полным перечнем операторов на сайте [rxmarbles](http://rxmarbles.com/).

## Scheduler
Очень важная тема, которую в этой статье я бы хотел затронуть, это scheduler. Scheduler, позволяют нам запускать наши observable на определенных потоках и в них есть свои тонкости. Начнем, существует 2 вида установить observable scheduler -  [observeOn]() и [subscribeOn]().
### SubscribeOn 
SubscribeOn отвечает за то, в каком потоке будет выполняться весь процесс observable до того момента, как его event'ы дойдут до обработчика (подписчика).
### ObserveOn 
Как можно догадаться observeOn отвечает за то, в каком потоке будут обрабатываться принятые подписчиком event'ы.  
Это очень крутая штука, потому что мы можем очень легко поставить загрузку чего либо из сети в background поток, а при получении результат перейти в main поток и как-то воздействовать на UI.

Давайте посмотрим, как это работает на примере:
```swift
let observable = Observable<Int>.create { (observer) -> Disposable in
    print("thread observable -> \(Thread.current)")
    observer.onNext(1)
    observer.onNext(2)
    return Disposables.create()
}.subscribeOn(ConcurrentDispatchQueueScheduler(qos: .background))

_ = observable
    .observeOn(MainScheduler.instance)
    .subscribe({ (e) in
        print("thread -> \(Thread.current)")
        print(e)
})
```

В консоль мы получим:
```
thread observable -> <NSThread: 0x604000465040>{number = 3, name = (null)}
thread -> <NSThread: 0x60000006f6c0>{number = 1, name = main}
next(1)
thread -> <NSThread: 0x60000006f6c0>{number = 1, name = main}
next(2)
```
Мы видим, что observable создавался в background потоке, а обрабатывали данные мы в main потоке. Это полезно при работе с сетью к примеру:
```swift
let rxRequest = URLSession.shared.rx.data(request: URLRequest(url: URL(string: "http://jsonplaceholder.typicode.com/posts/1")!)).subscribeOn(ConcurrentDispatchQueueScheduler(qos: .background))

_ = rxRequest
    .observeOn(MainScheduler.instance)
    .subscribe { (event) in
        print("данные \(event)")
        print("thread \(Thread.current)")
}
```

Таким образом запрос будет выполняться в background потоке, а вся обработка ответа будет происходить в main. На данном этапе пока рано говорить, что за _rx_ метод у _URLSession_ нарисовался вдруг, это будет рассмотренно позднее, данный код был приведен в качестве примера использования _Scheduler_, кстати, в консоль мы получим следующий ответ.

```
curl -X GET 
"http://jsonplaceholder.typicode.com/posts/1" -i -v
Success (305ms): Status 200
**данные next(292 bytes)**
thread -> <NSThread: 0x600000072580>{number = 1, name = main}
данные completed
thread -> <NSThread: 0x600000072580>{number = 1, name = main}
```

В финале давайте посмотрим еще что за _data_ нам пришла, для этого придется выполнить проверку, чтобы не начать парсить сообщение completed случайно.

```swift
_ = rxRequest
    .observeOn(MainScheduler.instance)
    .subscribe { (event) in
        if (!event.isCompleted && event.error == nil) {
            let json = try? JSONSerialization.jsonObject(with: event.element!, options: [])
            print(json!)
        }
        print("data -> \(event)")
        print("thread -> \(Thread.current)")
}
```
Мы проверяем, что event не сообщение о завершении работы observable и не ошибка пришедшая от него, хотя можно было реализовать другой метод подписки и обработать все эти виды event'ов отдельно, но это вы уже сможете сделать самостоятельно, а в консоль мы получим следующее.
```
curl -X GET 
"http://jsonplaceholder.typicode.com/posts/1" -i -v
Success (182ms): Status 200
{
    body = "quia et suscipit\nsuscipit recusandae consequuntur expedita et cum\nreprehenderit molestiae ut ut quas totam\nnostrum rerum est autem sunt rem eveniet architecto";
    id = 1;
    title = "sunt aut facere repellat provident occaecati excepturi optio reprehenderit";
    userId = 1;
}
data -> next(292 bytes)
thread -> <NSThread: 0x60400006c6c0>{number = 1, name = main}
data -> completed
thread -> <NSThread: 0x60400006c6c0>{number = 1, name = main}
```
Данные получены :-)

## Subjects 
Переходим к горячему, а именно от "холодных" или "пассивных" observable к "горячим" или "активным" observable, которые зовутся subject'ами. Если до этого наши observable начинали свою работу только после подписки на них и у вас был вопрос в голове "ну и зачем мне все это надо?", то Subject'ы работают всегда и всегда шлют полученные данные. 
Как это? В случае с observable мы заходили в поликлинику, шли к злой бабульке на ~~ресепшене~~ регистратуре, подходили и спрашивали в какой кабинет нам идти, тогда бабуленция нам отвечала, в случае с subject'ами, бабуленция стоит и слушает расписание и состояние врачей по больнице и как только получает информацию о перемещении какого-либо врача сразу говорит это, спрашивать у бабуленции что-то бесполезно, мы можем просто подойти, послушать ее, уйти, а она продолжит говорить, что-то увлекся со сравнениями, давайте уже к коду.
Создадим один subject и 2 подписчиков, первого создадим сразу после subject'а, пошлем subject'у значение, а потом создадим второго и пошлем еще парочку значений.
```swift
let subject = PublishSubject<Int>()
        
subject.subscribe { (event) in
    print("первый подписчик \(event)")
}
        
subject.onNext(1)
        
_ = subject.subscribe { (event) in
    print("второй подпичик \(event)")
}
        
subject.onNext(2)
subject.onNext(3)
subject.onNext(4)
```

Что мы увидим в консоли? правильно, первый успел получить первый event, а второй нет.
```
первый подписчик next(1)
первый подписчик next(2)
второй подпичик next(2)
первый подписчик next(3)
второй подпичик next(3)
первый подписчик next(4)
второй подпичик next(4)
```

Уже больше подходит под ваше представление о реактивном программировании?
Subject'ы бывают нескольких видов, все они отличаются тем, как они шлют значения.  

PublishSubject - самый простой, ему без разницы на все, он просто рассылает всем подписчикам то, что ему пришло и забывает об этом.  

ReplaySubject - а вот это самый ответственный, при создании мы указываем ему размер буфера (сколько значений будет запоминать), в результате он хранит в памяти последние n значений и посылает их сразу новому подписчику.  


```swift
let subject = ReplaySubject<Int>.create(bufferSize: 3)
        
subject.subscribe { (event) in
    print("первый подписчик \(event)")
}
        
subject.onNext(1)
        
subject.subscribe { (event) in
    print("второй подписчик \(event)")
}
        
subject.onNext(2)
subject.onNext(3)
        
subject.subscribe { (event) in
    print("третий подписчик \(event)")
}
        
subject.onNext(4)
```

Смотрим в консоль
```
первый подписчик next(1)
второй подписчик next(1)
первый подписчик next(2)
второй подписчик next(2)
первый подписчик next(3)
второй подписчик next(3)
третий подписчик next(1)
третий подписчик next(2)
третий подписчик next(3)
первый подписчик next(4)
второй подписчик next(4)
третий подписчик next(4)
```
BehaviorSubject - не такой пофигист, как предыдущий, он имеет стартовое значение и он запоминает последнее значение и посылает его сразу после подписки подписчика.
```swift
let subject = BehaviorSubject<Int>(value: 0)
        
subject.subscribe { (event) in
    print("первый подписчик \(event)")
}
        
subject.onNext(1)
        
subject.subscribe { (event) in
    print("второй подписчик \(event)")
}
        
subject.onNext(2)
subject.onNext(3)
        
subject.subscribe { (event) in
    print("третий подписчик \(event)")
}
        
subject.onNext(4)
```
Консоль
```
первый подписчик next(0)
первый подписчик next(1)
второй подписчик next(1)
первый подписчик next(2)
второй подписчик next(2)
первый подписчик next(3)
второй подписчик next(3)
третий подписчик next(3)
первый подписчик next(4)
второй подписчик next(4)
третий подписчик next(4)
```

## Заключение
Это была вводная статья, написанная для того, чтобы вы знали основы и могли в последующем отталкиваться от них.  В следующих статьях мы рассмотрим как работать с помощью RxSwift с UI компонентами iOS,  создание расширений для UI компонентов.

### Не RxSwift'ом едины
Реактивное программирование реализовано не только в библиотеке RxSwift, есть несколько реализаций, вот самые популярные из них [ReacktiveKit/Bond](), [ReactiveSwift/ReactiveCocoa](). У всех у них есть небольшие различия в реализации под капотом, но на мой взгляд лучше начинать свое познание "реактивщины" именно с RxSwift, так как он наиболее популярный среди них и по нему будет больше ответов в великом [гугле](https://google.com/), но после того, как вы вникните в суть данной концепции, можно выбирать библиотеки на свой вкус и цвет.

