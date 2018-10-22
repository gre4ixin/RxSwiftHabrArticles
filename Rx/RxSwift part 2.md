![ReactiveX logo](http://www.pvsm.ru/images/2015/12/19/realizaciya-MVVM-v-iOS-s-pomoshyu-RxSwift.jpg "ReactiveX")


Всем доброго времени суток. Начнем нашу вторую статью цикла посвященного функционально-реактивному программированию в iOS.
В прошлой статье мы познакомились с observable, subscriber, небольшим количеством операторов и subject'ами. Но представление о том, как это использовать в бою все равно слабое. Сегодня мы разберемся как работать с UI, писать расширения для каких-то свойств UI компонентов и немного поговорим про архитектуры.

</cut>

### UI
За UI отвечает второй фреймворк с названием RxCocoa. Поэтому для начала надо сделать его импорт в наш ViewController.

```swift
import RxCocoa
```

Теперь давайте реализуем самый популярный пример, который приводится для демонстрации "реактивщины", это bind'инг textField'а к label'у. То есть мы связываем 2 UI элемента и говорим, что хотим видеть все, что есть textField'е, в нашем label'е. Для этого надо набросать эти элементы на наш storyboard, привязать их к контроллеру и приступить к связке.
```swift
@IBOutlet var textField: UITextField!
@IBOutlet var label: UILabel!

self.textField.rx.text
    .bind(to: self.label.rx.text)
    .disposed(by: bag)    
```

![NotBad mem](https://pbs.twimg.com/media/ClsAM-CUYAIGR9V.png:large)

И все, теперь запустите проект и попробуйте повводить что-нибудь в textField, вы сразу увидите изменения в вашем label'е. Удобно? Мне кажется, что да.
#### Что такое bind
Bind создает новую подписку и отправляет элементы наблюдателю, bind был сделан для упрощения написания/чтения и понимания реактивного кода.
Чтобы не писать 
```swift
self.textField.rx.text.subscribe { (event) in
// проверки, что event не error и не completed
    self.label.text = event.element!
}.disposed(by: disposeBag)
```
Слишком большая запись, поэтому был придуман **bind**, с ним все выглядит гораздо более наглядно и лаконично. Работает под капотом это тоже несложно, **Binder** это структура, которая наследуется от `ObserverType` и принимает в качестве значения все что угодно, дженерик проще говоря, чуть позже рассмотрим, что там творится под капотом.
```swift
self.textField.rx.text.bind(to: self.label.rx.text)
    .disposed(by: disposeBag)
```
Согласитесь, что так выглядит гораздо лучше, правда? В эту конструкцию мы можем встраивать любые операции, рассмотренные в прошлой статье и имеющиеся в RxSwift. Правда, не все делается так легко и в одну строчку, например работа с таблицей потребует от нас логику создания ячейки, эдакий **cellForRow AtIndexPath**, но в исполнении RxSwift.

#### TableView

Для работы с таблицей у нас есть методы, встроенные в RxCocoa.

Все, что нужно сделать, чтобы наполнить таблицу данными это привязать ее к какому нибудь Observable или Subject.

```swift
let observableForTableViewTest = Observable<String>.of("Кошка", "Собака", "Гусь", "Курица")
        
observableForTableViewTest
    .bind(to: self.tableView.rx.items(cellIdentifier: "Cell")) { index, element, cell in
    //настраиваем нашу ячейку
}.disposed(by: disposeBag)
```

Всё остальное выполнится само, из колличества элементов в Observable будет высчитано количество ячеек, а если вы подпишетесь на subject, то каждый раз посылая новый event, таблица будет обрабатывать его (добавлять, удалять ячейку) и сама перезагружаться. Ваша задача создать связку и указать кто от кого зависит и кто что делает при том или ином событии. У нас также есть другие методы делегата таблицы.

```swift
self.tableView.rx.willDisplayCell.subscribe { event in
    //обабатываем ячейку            
}.disposed(by: disposeBag)
```
Event в данном случае хранит ячейку и ее индекс

```swift
event.element?.cell
event.element?.indexPath
```
Есть для обработки нажатия на ячейку

```swift
self.tableView.rx.itemSelected.subscribe { event in
    event.element?.item //Индекс ячейки
}.disposed(by: disposeBag)
```

В RxCocoa есть реализация большинства методов и свойст UI элементов в реактивном стиле, но бывают случаи, когда мы используем что-то кастомное или какого-то свойства просто нет в RxCocoa, а нам оно очень нужно и не хочется писать в ином стиле среди всей реактивщины, тогда нам на помощь приходите ReactiveExtension.

#### ReactiveExtension
ReactiveExtension позволяет нам создать расширение для любого элемента и любого его свойства реактивный аналог, в который мы сможем "биндить" все, что нам нужно.
Например мы создаем WKWebView, пишем у него rx и видим, что доступных методов там нет. Что делать? 
Создать Extension, это очень просто, допустим мы имеем  Observable, который кидает URLRequest, и мы не хотим делать следующее 
```swift
let request = URLRequest(url: URL(string: "")!)
let observable = Observable<URLRequest>.just(request)

observable.subscribe { [unowned self] request in 
    self.webView.load(request.element!)
}.disposed(by: disposeBag)
```
А еще надо проверить не пустой ли ответ... это все долго, я ленивый, я хочу примерно вот так "obs.bind(to: self.webView.load)" и все, потому что в случае с подпиской надо не забыть про `[unowned self]`, выполнить проверки, это много кода... но через `bind(to:)` не получается, значит надо сделать так, чтобы получилось. Делается это очень просто
```swift
extension Reactive where Base: WKWebView {
    
    public var loadRequest: Binder<URLRequest> {
        return Binder(self.base) { webView, request in
            webView.load(request)
        }
    }   
}
```

Вот и все, теперь если мы выполним 
```swift
let observable = Observable<URLRequest>.just(request)
observable.bind(to: self.webView.rx.loadRequest)
    .disposed(by: disposeBag)
```
то все будет работать, как мы хотим. Чтобы создать Extension, нам необходимо создать расширение для Reactive (структура, если посмотреть что это такое), сказать, что Base должно быть WKWebView, создать переменную, обозвать ~~её как нам вздумается~~ , чтобы из названия было понятно, что произойдет, далее мы должны сказать, что наша перменная это Binder с типом URLRequest и вернуть нам Binder, который в функции подставляет необходимые значения в нужные места обычного метода у webView. На самом деле, таким образом устроены все UI элементы в RxCocoa, если вы откроете какой-нибудь Label или TextField, то увидите там то же самое, с разницей в том, что Binder может меняться на ControlProperty. ControlProperty как правило возвращает нам значение какого-то свойства или события (tap по кнопке), а Binder используется для bind'а. Если вам трудно запомнить синтаксис расширений, то можно выбирать похожую реализацию у RxCocoa и копировать ее, изменяя под себя. Теперь надо немного поговорить об `[unowned self]`.
#### [unowned self]
В iOS используется ARC для управления памятью, то есть подсчет ссылок Automatic reference counting. Это значит, что объект живет до тех пор, пока на него ссылается хотя бы одна strong(сильная) ссылка. Когда мы в closure подписчика вызываем self и берем что-то, то мы держим наш модуль, который в свою очередь держит нас, такая связь не прервется, потому что они не смогут уже отпустить друг друга и произойдет утечка памяти. 

Используя `[unowned self]`, мы говорим, чтобы self не удерживала объект. Можно также использовать `[weak self]`, но тогда `self` будет со знаком вопроса и выглядит это не красиво, но все же есть случаи, когда `[weak self]` необходим, в случае например, когда self у нас может в какой-то момент оказаться nil, в остальных же случаях можно использовать unowned.

#### Архитектуры
Работать с UI в RxCocoa достаточно легко, главное понять принципы описанные в первой статье и посмотреть пару примеров в этой статье, а дальше эксперименты в playground или чистом проекте. Я бы хотел поведать дальше про архитектуры, которые позволяют использовать RxCocoa/RxSwift более продуктивно и удобно. Это так называемые однонаправленные архитектуры или flux-архитектуры. У таких подходов есть как плюсы, так и минусы, поэтому необходимо для начала все взвесить.

#### Flux-архитектуры
Использовать однонаправленные архитектуры в связе с реактивом очень удобно. В чем их суть.  

##### State
У нас имеется реактивный State, то место, где мы храним все свойства, нам не надо заморачиваться и вешать каждый раз по `observable` или `subject` на каждое свойство, мы создаем свойства как и прежде, сам State лежит в `Subject'е` и когда мы что-то поменяли в нем, то он разошлется всем подписчикам в обновленном состоянии.

##### Reducer
В однонаправленных архитектурах есть лишь одно место, где можно изменить свойства `State'а` и я считаю это тоже плюсом, не выйдет поменять `State` в контроллере, все изменения происходят в одном месте и очень уднобно ловить ошибки.

##### Очередь
Все события отправленные в `reducer` встают в очередь и выполняются по очереди, это тоже удобно, все действия выполнятся по порядку и не получится так, что два разных действия поработают с одними и теми же данными. 

##### Middleware
Могут называться необязательно так, но суть одна, имеются механизмы, которые позволяют перехватить какое-либо событие и видоизменить его, это можно быть в момент отправки, в момент обработки события или после обработки.

Это основные модули, которые имеются в реализациях такой архитектуры, также видел реализации, где был добавлен еще роутинг по аналогии с reducer, но это уже детали, давайте взглянем на одну из таких реализаций под названием [ReactorKit](https://github.com/ReactorKit/ReactorKit)

#### ReactorKit
Это достаточно простая реализация.

Создадим экран, с двумя кнопками и лейблом, в лейбле поумолчанию будет стоять значение 0, по нажатию на одну из кнопок с названием "добавить" будем прибавлять в лейбле единицу, по нажатию на кнопку "отнять", будем вычитать оттуда единицу, поехали.

Создадим обычный контроллер

```swift
class ReactorViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
    }
}
```

Теперь нам надо подписаться на протокол нашей библиотеки, называется он `StoryboardView`. Когда мы подпишем наш контроллер на этот протокол, то от нас потребуют реализовать метод `bind` и добавить свойство `disposeBag`. Данная библиотека использует [method swizzling](https://nshipster.com/method-swizzling/), если коротко, то это подмена каких-то методов в данном случае метода `viewDidLoad` у контроллера, на какую-то кастомную реализацию, тут подменяется на viewDidLoad с вызовом метода `bind`, поэтому его можно вообще не вызывать, он сам сработает, это минус как по мне, `method swizzling` может доставить много неприятностей и надо быть с ним очень осторожным.

Теперь нам надо создать еще один Swift файл, пустой, в котором мы опишем логику ReactorKit'а. 

В созданном файле создаем класс и подписываемся на протокол `Reactor`.

В этом классе нам надо создать два `enum'а` это `Action`  и `Mutation`. `Action` отвечает за посылку каких-либо действий пользователя, `Mutation` за работу со `State'ом`. Например, мы нажимаем на кнопку "добавить" и должны отправить Action "addButtonTapped",  Mutation в данном случае это действия со State'ом и его мы назовем increment(название нашего свойства).

Теперь создадим `State`, состояние у нас является структурой, в нем буду храниться наши свойства. Как выглядит наш класс после создания State, Action и Mutation.
```swift
import ReactorKit
import RxSwift

class HabrReactorView: Reactor {
    
    enum Action {
        case incrementButtonTapped
        case decrimentButtonTapped
    }
    
    enum Mutation {
        case incrementValue
        case decrimentValue
    }
    
    struct State {
        var reactValue: Int
    }
    
}
```

Теперь добавим свойство `initialState`, которое требует от нас протокол `ReactorView` и реализуем метод `init`.

```swift
var initialState: State
    
init() {
    initialState = State(reactValue: 0)
}
```

Осталось реализовать 2 метода, первый это тот, где мы будем делать обработку `Action'ов` и возвращаться `Mutation`, а второй где мы будем изменять наш `State` и возвращать его измененный вид.

```swift
func mutate(action: Action) -> Observable<Mutation> {
    switch action {
    case .incrementButtonTapped:
        return Observable.just(Mutation.incrementValue)
    case .decrimentButtonTapped:
        return Observable.just(Mutation.decrimentValue)
    }
}
    
func reduce(state: State, mutation: Mutation) -> State {
    var newState = state
    switch mutation {
    case .incrementValue:
        newState.reactValue += 1
    case .decrimentValue:
        newState.reactValue -= 1
    }
    return newState
}
```

Когда происходит какое-то событие, то мы посылаем `Action`, наш `action` попадает в функцию `mutate`, где `switch'ом` мы проходим по всем `action'ам` в поисках нашего `action'а` и возвращаем `Observable` с enum'ом `Mutation'а`. После этого наш `Mutation enum` попадет в функцию `reduce`, в которой мы копируем наш `State` и точно также проходим `switch'ом` по всем enum'ом, выполняя определенные действия над состоянием. 

Плюс в том, что мы не сможем изменить `State`, в контроллере не прибегая к отправке `Action'а`, таким образом мы сможем отлавливать все изменения в состоянии в одном месте и вся логика работы с состоянием находится в том же месте. Вернемся к нашей задаче.

Мы описали `Reactor` и теперь надо вернуться в контроллер и сделать там привязку, полагаю, что создать кнопки (не `@IBAction`, а `@IBOutlet`), с лейблом вы сможете самостоятельно, поэтому переходим в функцию `bind`, во входные параметры добавляем наш Reactor, а внутри делаем привязку. Нам надо привязать нажатия на кнопки к нужным `Action'ам`, а лейблу следить за состоянием и запихивать туда то, что находится в `State`.

```swift
func bind(reactor: HabrReactorView) {
  
  // по нажатию на кнопку отправляем экшн
  incrementButton.rx.tap.map { Reactor.Action.incrementButtonTapped }
        .bind(to: reactor.action)
        .disposed(by: disposeBag)
  // по нажатию на кнопку отправляем экшн
  decrimentButton.rx.tap.map { Reactor.Action.decrimentButtonTapped }
        .bind(to: reactor.action)
        .disposed(by: disposeBag)
  //подписываемся на состояние, в пришедшем состоянии вытаскиваем нужное свойство, конвертируем в строку и отдаем нашему лейблу
  reactor.state.map { String($0.reactValue) }
        .bind(to: label.rx.text)
        .disposed(by: disposeBag)
}
```

Для завершения осталось сделать последний штрих, это в AppDelegate запихнуть переход к нашему контроллеру, можете убрать галочку у контроллера в `storyboard'е` "is initial viewcontroller", мы будем добавлять его руками. Переходим в `AppDelegate`, функция `didFinishLaunchingWithOptions`. 

```swift
var window: UIWindow?


func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

    self.window = UIWindow(frame: UIScreen.main.bounds)
        
    let reactViewController = UIStoryboard(name: "HabrTest", bundle: nil).instantiateViewController(withIdentifier: "HabrReactorViewController") as! HabrReactorViewController
    //не забываем, что надо заинжектить Reactor
    reactViewController.reactor = HabrReactorView()
        
    self.window?.rootViewController = UINavigationController(rootViewController: reactViewController)
    self.window?.makeKeyAndVisible()
        
    return true
}
```

Готово, теперь можно запускать и тестировать.

##### Middleware
Мы можем перехватить `Action` или `Mutation` перед его выполнением и изменить, делается это с помощью функций transform, вернемся в файл `HabrReactorView` и в классе реализуем функции. Я хочу к примеру, чтобы во время нажатия кнопки "Отнять" срабатывал бы эффект "Добавить", а у кнопки  наоборот эффект отнять, вроде в данном случае это просто пример, но в реальной жизни нам может понадобиться проверять все `Action'ы` на наличие интернет соединения допустим, если сети нет, то показывать ошибку, а `Action'ов` у нас может быть гораздо больше, чем два.

```swift
func transform(action: Observable<Action>) -> Observable<Action> {
    let newAction = action.map { (event) -> Action in
        if (event == Action.incrementButtonTapped) {
            return Action.decrimentButtonTapped
        } else {
            return Action.incrementButtonTapped
        }
    }
    return newAction
}
```

В контроллере мы ничего не меняем, просто делаем билд и все готов. В данной связке мы перед тем как отправить наш `Action` в путешествие к функции `reduce` мы попадаем сюда, делаем `map'ом` проверку на то, что этот `Action` добавляет значение и отправляем назад обратный ему `Action`. То же самое можно делать и с `Mutation`. 

Теперь когда мы освоились немного с системой, попробуем написать что-то более реальное, например такое, возьмем сайт [jsonplaceholder](http://jsonplaceholder.typicode.com), будем делать к нему запрос https://jsonplaceholder.typicode.com/posts/1 и кадый раз будем прибавлять по единице на параметр и добавлять новый результат в таблицу. Предварительно само собой все будем парсить в модельку. Для работы с сетью будем использовать [RxAlamofire](https://github.com/RxSwiftCommunity/RxAlamofire), для этого допишем в наши поды ее и сделаем `install`.

Приступим, проделаем все операции по созданию контроллера и реактора, добавим на сторибор таблицу и под ней кнопку, привяжем все это к коду и начнем пробовать.

Для начала создадим модель, куда будем парсить наши данные.

```swift
struct ResponseObject: Codable, Equatable {
    var userId: Int
    var title: String
    var body: String
    var id: Int
}
```

`Equatable` нам понадобится позже, чтобы мы не обновляли таблицу при каждом обновлении State.

В Swift'е в `enum'е` можно передавать значение, надеюсь вы это знаете, поэтому мы будем кидать инфу через наши `enum'ы`.

```swift
enum Action {
    // actiom cases
    case addButtonTapped
}

enum Mutation {
    // mutation cases
    case showActivity, hideActivity
    
    case successWithData(ResponseObject), errorRequest
}
```

Получается в Action нам будет приходит только один `enum`, это то что кнопка нажата, а в Mutation будет происходить побольше событий.

Создадим пока функцию запроса, в нее будет передаваться id а возвращаться `Observable<Mutation>` либо success, либо error.

```swift
//запросы шлем с помощью Alamofire, поэтому надо ее добавить в импорт
import RxAlamofire

//MARK: - Private
private func makeRequest(with identifier:Int) -> Observable<Mutation> {
  
  //делаем проверку, что наш id больше 0 и меньше 100 (больше нет записей на сервере)
    let currentId = (identifier <= 0) && (identifier > 100) ? 1 : identifier
    
    //возвращаем observable
    return requestData(.get, "\(baseUrl)\(currentId)").map({ (response) -> Mutation in
    
    //если статус код 404, то кидаем ошибку
        if (response.0.statusCode == 404) {
            return Mutation.errorRequest
        } else {
            let response = try! JSONDecoder().decode(ResponseObject.self, from: response.1)
            return Mutation.successWithData(response)
        }
    })
}
```
В функции `mutate` мы объединяем наши Observable в один с помощью `concat`.
```swift
func mutate(action: Action) -> Observable<Mutation> {
     switch action {
     case .addButtonTapped:
        return Observable.concat([
            Observable.just(Mutation.showActivity),
            // currentState это просто структура текущего состояния State'а не путать с переменной "state"
            makeRequest(with: self.currentState.currentIdentifier + 1),
            Observable.just(Mutation.hideActivity)
            ])
     }
}
```

В функции `reduce` пишем логику работы со `state'ом`.

```swift
func reduce(state: State, mutation: Mutation) -> State {
    var newState = state
     switch mutation {
     case .showActivity:
        newState.load = true
     case .hideActivity:
        newState.load = false
     case .errorRequest:
        newState.error = true
     case .successWithData(let data):
        var items = state.responseObject
        items.append(data)
        newState.responseObject = items
    }
    return newState
}
```

И сам State у нас будет выглядеть так

```swift
struct State: Equatable {
    
    //state
    var load: Bool = false
    var responseObject: [ResponseObject] = []
    var currentIdentifier: Int = 0
    var error: Bool = false
    
    static func == (lhs: JsonPlaceholerTestReactorView.State, rhs: JsonPlaceholerTestReactorView.State) -> Bool {
        return lhs.responseObject == rhs.responseObject
    }
}
```

Теперь нам надо добавить еще один `enum`, чтобы увеличивать счетчик `currentIdentifier`.

```swift
enum Mutation {
        // mutation cases
    case showActivity, hideActivity
    case updateIdentifier(Int)
        
    case successWithData(ResponseObject), errorRequest
}
```

Добавим его только в `Mutation` и вызовем его в Observable, теперь наша функция `mutate` и `reduce` будет выглядеть так.

```swift
func mutate(action: Action) -> Observable<Mutation> {
     switch action {
     case .addButtonTapped:
        return Observable.concat([
            Observable.just(Mutation.showActivity),
            Observable.just(Mutation.updateIdentifier(self.currentState.currentIdentifier + 1)),
            makeRequest(with: self.currentState.currentIdentifier + 1),
            Observable.just(Mutation.hideActivity)
                                  ])
     }
}

func reduce(state: State, mutation: Mutation) -> State {
    var newState = state
     switch mutation {
     case .showActivity:
        newState.load = true
     case .hideActivity:
        newState.load = false
     case .updateIdentifier(let identifier):
        newState.currentIdentifier = identifier
     case .errorRequest:
        newState.error = true
     case .successWithData(let data):
        var items = state.responseObject
        items.append(data)
        newState.responseObject = items
    }
    return newState
}
```

Теперь идем в контроллер и начинаем привязывать все это дело к UI.

```swift
nextPostButton.rx.tap.map { Reactor.Action.addButtonTapped }.bind(to: reactor.action).disposed(by: disposeBag)

reactor.state.map { $0.load }.bind(to: activityIdicator.rx.isAnimating).disposed(by: disposeBag)
reactor.state.map { !$0.load }.bind(to: activityIdicator.rx.isHidden).disposed(by: disposeBag)

//table view
reactor.state.map { $0.responseObject }.distinctUntilChanged().bind(to: tableView.rx.items(cellIdentifier: "Cell")) { (index, element, cell) in
    cell.textLabel?.text = element.title
}.disposed(by: disposeBag)
```

У таблицы добавим вызов функции `distinctUntilChanged`, это означает что если данные пришли такие же, как в прошлый раз, то обновляться таблица не будет.

Мы также добавили `activityIndicator`, чтобы пока мы ждем ответ от сервера он показывал, что идет загрузка.

Последним примером давайте напишем поиск людей вконтакте с использованием VK API.

