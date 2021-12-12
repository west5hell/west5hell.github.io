---
layout:     post
title:      RxSwift Combining Operators（组合操作符）
date:       2021-12-12
categories: RxSwift
---

# Combining Operators（组合操作符）

* ## Prefixing and concatenating（前缀和连接）

## 1. `startWith`

在可观察序列前面加上给定的初始值。此值必须与可观察元素的类型相同
```
let numbers = Observable.of(2,3,4)
    
let observable = numbers.startWith(111)
    
_ = observable.subscribe(onNext: { value in
    print(value)
})
```

输出：
```
111
2
3
4
```

## 2. `Observable.concat`

连接两个可观察序列

```
let first = Observable.of(1,2,3)
let second = Observable.of(4,5,6)
    
let observable = Observable.concat([first, second])
    
observable.subscribe(onNext: {
    print($0)
})
```

输出：
```
1
2
3
4
5
6
```

## 3. `concat`

```
let germanCities = Observable.of("Berlin", "Munich", "Frankfurt")
let spanishCities = Observable.of("Madrid", "Barcelona", "Valencia")
    
let observable = germanCities.concat(spanishCities)
    
_ = observable.subscribe(onNext: {
    print($0)
})
```

输出：
```
Berlin
Munich
Frankfurt
Madrid
Barcelona
Valencia
```

## 4. `concatMap`

> `concatMap(_:)`与「转换操作符」中的`flatMap(_:)`密切相关。你传递给`concatMap(_:)`的闭包返回一个 Observable 序列，操作符首先订阅该序列，然后将其发出的值转发到产生的序列中。`concatMap(_:)`保证闭包产生的每个序列在订阅下一个序列之前都运行完毕。这是一个保证顺序的便捷方式，同时给你带来了`flatMap(_:)`的力量。

```
let sequences = ["German cities": Observable.of("Berlin", "Münich",
                                                "Frankfurt"),
                 "Spanish cities": Observable.of("Madrid", "Barcelona",
                                                 "Valencia")]
    
let observable = Observable.of("German cities", "Spanish cities")
    .concatMap { country in sequences[country] ?? .empty() }
    
_ = observable.subscribe(onNext: {
    print($0)
})
```

输出：

```
Berlin
Münich
Frankfurt
Madrid
Barcelona
Valencia
```

* ## Merging

## 1. `merge`

![merge](https://i.v2ex.co/cjUt1tJ4l.png)


> `merge()`可观察对象订阅它接受的每个序列，并在元素到达后立即发出元素 - 没有预定的顺序。
> `merge()`在其源序列完成且所有内部序列完成后完成。
> 内部序列序列完成的顺序无关紧要。
> 如果任何序列发出错误，`merge()`可观测值会立即中继错误，然后终止。

```
let left = PublishSubject<String>()
let right = PublishSubject<String>()
    
let source = Observable.of(left.asObservable(), right.asObservable())
    
let observable = source.merge()
    
_ = observable.subscribe(onNext: {
    print($0)
})
    
var leftValues = ["Berlin", "Munich", "Frankfurt"]
var rightValues = ["Madrid", "Barcelona", "Valencia"]
repeat {
    switch Bool.random() {
    case true where !leftValues.isEmpty:
        left.onNext("Left: " + leftValues.removeFirst())
    case false where !rightValues.isEmpty:
        right.onNext("Right: " + rightValues.removeFirst())
    default:
        break
    }
} while !leftValues.isEmpty || !rightValues.isEmpty
    
left.onCompleted()
right.onCompleted()
```

输出：

```
Left: Berlin
Right: Madrid
Left: Munich
Left: Frankfurt
Right: Barcelona
Right: Valencia
```

* ## Combining elements（组合元素）

## 1. `combineLatest`

![combineLatest](https://i.v2ex.co/4ki20GNGl.png)

> 每当其中一个内部序列发出一个值时，它都会调用您提供的闭包。您将接收到每个内部序列发出的最后一个值。这有许多具体的应用，例如一次观察多个文本字段并组合其值，监视多个源的状态等。


```
let left = PublishSubject<String>()
let right = PublishSubject<String>()
    
let observable = Observable.combineLatest(left, right) { lastLeft, lastRight in
    "\(lastLeft) \(lastRight)"
}
    
_ = observable.subscribe(onNext: {
    print($0)
})
    
print("> Sending a value to Left")
left.onNext("Hello,")
print("> Sending a value to Right")
right.onNext("world")
print("> Sending another value to Right")
right.onNext("RxSwift")
print("> Sending another value to Left")
left.onNext("Have a good day,")
    
left.onCompleted()
right.onCompleted()
```

输出：

```
> Sending a value to Left
> Sending a value to Right
Hello, world
> Sending another value to Right
Hello, RxSwift
> Sending another value to Left
Have a good day, RxSwift
```

> 请记住`combineLatest(_:_:resultSelector:)`等待其所有可观察对象发出一个元素，然后再开始调用您的闭包。这是一个经常引起混淆的根源，也是使用`startWith(_:)`的好机会，这些序列可能不会立即传递出一个值。

一种常见的模式是将值组合到元组，然后将它们传递到链中。例如，您经常需要合并值，然后调用 `filter(_:)`在他们身上，就像这样：

```
let observable = Observable
    .combineLatest(left, right) { ($0, $1) }
    .filter { !$0.0.isEmpty }
```

在`combineLatest`系列运算符中有几个变体。它们采用两到八个可观察序列作为参数。序列不需要具有相同的元素类型。

```
let choice: Observable<DateFormatter.Style> = Observable.of(.short, .long)
let dates = Observable.of(Date())
    
let observable = Observable.combineLatest(choice, dates) { format, when -> String in
    let formatter = DateFormatter()
    formatter.dateStyle = format
    return formatter.string(from: when)
}
    
_ = observable.subscribe(onNext: { value in
    print(value)
})
```

输出：

```
12/12/21
December 12, 2021
```

## 2. `zip`

```
enum Weather {
    case cloudy
    case sunny
}
    
let left: Observable<Weather> = Observable.of(.sunny, .cloudy, .cloudy, .sunny)
let right = Observable.of("Lisbon", "Copenhagen", "London", "Madrid", "Vienna")
    
let observable = Observable.zip(left, right) { weather, city in
    return "It's \(weather) in \(city)"
}
    
_ = observable.subscribe(onNext: {
    print($0)
})
```

输出：

```
It's sunny in Lisbon
It's cloudy in Copenhagen
It's cloudy in London
It's sunny in Madrid
```

* 订阅您提供的可观察量。
* 等待每个都发出新值。
* 使用两个新值调用您的闭包。

> 你有没有注意到 Vienna 没有出现在输出中？为什么？ 

> 原因在于 zip 运算符的工作方式。它们将每个可观察量的每个下一个值配对在相同的逻辑位置（1st与1st，2nd与2nd等）。这意味着，如果其中一个内部可观察量的下一个值在下一个逻辑位置不可用（即，因为它已完成，如上例所示），则 zip 将不再发出任何内容。这称为索引序列，这是一种同步运行序列的方法。但是，尽管 zip 可能会提前停止发出值，但在所有内部可观察对象完成之前，它本身不会完成，从而确保每个可观察对象都可以完成其工作。

* ## `Triggers`（触发器）

1. `withLatestFrom(_:)`

```
let button = PublishSubject<Void>()
let textField = PublishSubject<String>()
    
let observable = button.withLatestFrom(textField)

_ = observable.subscribe(onNext: {
    print($0)
})
    
textField.onNext("Par")
textField.onNext("Pari")
textField.onNext("Paris")
button.onNext(())
button.onNext(())
```

输出：

```
Paris
Paris
```

> 1. 创建两个模拟按钮点击和文本字段输入的主题。由于按钮 不携带真实数据，可以使用 Void 作为元素类型。 
> 2. 当按钮发出值时，请忽略它，而是发出收到的最新值 从模拟文本字段中。 
> 3. 模拟文本字段的连续输入，这是通过两个连续的按钮点击完成的。

2. `sample(_:)`

```
let button = PublishSubject<Void>()
let textField = PublishSubject<String>()
    
let observable = textField.sample(button)
_ = observable.subscribe(onNext: {
    print($0)
})
    
textField.onNext("Par")
textField.onNext("Pari")
textField.onNext("Paris")
button.onNext(())
button.onNext(())
```

输出：

```
Paris
```

> 每次触发器可观察量发出一个值时，`sample(_:)`从"other"可观察对象发出最新值，但前提是它自上次触发器以来到达。如果没有新数据到达，`sample(_:)`不会发出任何信息。

* ## Switches

## 1. `amb`

```
let left = PublishSubject<String>()
let right = PublishSubject<String>()
    
let observable = left.amb(right)
    
_ = observable.subscribe(onNext: {
    print($0)
})
    
left.onNext("Lisbon")
right.onNext("Copenhagen")
left.onNext("London")
left.onNext("Madrid")
right.onNext("Vienna")
    
left.onCompleted()
right.onCompleted()
```

输出：

```
Lisbon
London
Madrid
```

> 您会注意到，调试输出仅显示左侧主题中的项。以下是您所做的： 
> 1. 创建一个可观察对象，以解决左右两者之间的歧义。 
> 2. 让两个可观察量发送数据。 AMB（_：）运算符订阅左侧和右侧可观察量。它等待其中任何一个元素发出一个元素，然后取消订阅另一个元素。之后，它仅中继来自第一个活动可观察对象的元素。它确实从"模棱两可"这个词中得出了它的名字：起初，你不知道你对哪个序列感兴趣，只想决定什么时候触发。 此运算符经常被忽视。它有一些选择的实际应用程序，例如连接到冗余服务器并坚持使用首先响应的服务器。

2. `switchLatest`

```
let one = PublishSubject<String>()
let two = PublishSubject<String>()
let three = PublishSubject<String>()
    
let source = PublishSubject<Observable<String>>()
    
let observable = source.switchLatest()
let disposable = observable.subscribe(onNext: { value in
    print(value)
})
    
source.onNext(one)
one.onNext("Some text from sequence one")
two.onNext("Some text from sequence two")
    
source.onNext(two)
two.onNext("More text from sequence two")
one.onNext("and also from sequence one")
    
source.onNext(three)
two.onNext("Why don't you see me?")
one.onNext("I'm alone, help me")
three.onNext("Hey it's three. I win.")
    
source.onNext(one)
one.onNext("Nope. It's me, one!")
    
disposable.dispose()
```

输出：

```
Some text from sequence one
More text from sequence two
Hey it's three. I win.
Nope. It's me, one!
```

请注意输出，订阅只打印了推送到源可观察的最新序列中的项目。这就是`switchLatest()`的目的。

* ## Combining elements within a sequence（组合序列中的元素）

## 1. `reduce`

```
let source = Observable.of(1,3,5,7,9)
    
let observable = source.reduce(0, accumulator: +)
_ = observable.subscribe(onNext: {
    print($0)
})
```

输出：

```
25
```

## 2. `scan`

```
let source = Observable.of(1,3,5,7,9)
    
let observable = source.scan(0, accumulator: +)
_ = observable.subscribe(onNext: {
    print($0)
})
```

输出：

```
1
4
9
16
25
```