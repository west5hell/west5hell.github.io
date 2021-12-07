---
layout:     post
title:      RxSwift Transforming Operators（转换操作符）
date:       2021-12-07
categories: RxSwift
---

## Transforming Operators (转换操作符)

1. `toArray` 

将可观察的元素序列转换成这些元素的的数组，`toArray`操作符返回一个`Single`。

```
Observable.of("A", "B", "C", "D")
    .toArray()
    .subscribe(onSuccess: { array in
        print(array)
    }, onFailure: { _ in
        
    }, onDisposed: {
        
    })
    .disposed(by: disposeBag)
```
输出：
```
["A", "B", "C", "D"]
```

2. `map`

与Swift的标准`map`一样，只不过是对可观察变量进行操作。

```
let disposeBag = DisposeBag()
    
let formatter = NumberFormatter()
formatter.numberStyle = .spellOut
    
Observable<Int>.of(123, 4,56)
    .map {
        formatter.string(for: $0) ?? ""
    }
    .subscribe(onNext: {
        print($0)
    })
    .disposed(by: disposeBag)
```

输出：
```
one hundred twenty-three
four
fifty-six
```

`enumerated`和`map`组合使用
```
Observable.of(1,2,3,4,5,6)
    .enumerated()
    .map { index, integer in
        index > 2 ? integer * 2 : integer
    }
    .subscribe(onNext: {
        print($0)
    })
    .disposed(by: disposeBag)
```

输出：
```
1
2
3
8
10
12
```

3. `compactMap`

该操作符是`map`和`filter`操作符的组合，专门过滤掉`nil`值，类似于Swift标准库中的对应操作。

```
Observable.of("To", "be", nil, "or", "not", "to", "be", nil)
    .compactMap { $0 }
    .toArray()
    .map { $0.joined(separator: " ")}
    .subscribe(onSuccess: {
        print($0)
    })
    .disposed(by: disposeBag)
```
输出：
```
To be or not to be
```

#### Transforming inner observables（转换内部可观察量）

```
struct Student {
    let score: BehaviorSubject<Int>
}
```

RxSwift 在 `flatMap` 系列中包含一些运算符，允许您访问可观察对象并使用其可观察属性。

4. `flatMap`

![flatMap](https://i.v2ex.co/m2HKD17Bl.png)

观察此图中发生的情况的最简单方法是将每条路径从顶线上的源可观察路径一直到底线上的目标可观察路径，这将向订阅者提供元素。

源可观察量是具有 `value` 属性的对象类型，该属性本身是 `Int` 类型的可观察量。它的 `value` 属性的初始值是对象的编号，即 O1 的初始值为 1，O2 为 2，O3 为 3。

从 O1 开始，`flatMap` 接收对象，并将其值属性投影到 `flatMap` 下方第 1 行上为 O1 创建的新可观察对象上。然后，该可观察量被平展到底线上可观察的目标。

稍后，O1 的值属性将更改为 4。这在弹珠图中没有直观地表示。然而，O1的值已经改变的证据是，它被投影到O1的现有可观测值上，然后平展到目标可观察量。

源可观察量中的下一个值 O2 由 `flatMap` 接收。它的初始值 2 被投影到 O2 的新可观测值上，然后将其平展到目标可观测值。稍后，O2 的值更改为 5。然后，该值被投影并展平到目标可观察量。

最后，通过 `flatMap` 接收 O3，其初始值 3 被投影并展平。

回顾一下，`flatMap` 会投影并变换可观察量的可观测值，以及 然后将其平展为可观察的目标。

```
let laura = Student(score: BehaviorSubject(value: 80))
let charlotte = Student(score: BehaviorSubject(value: 90))
    
let student = PublishSubject<Student>()
    
student
    .flatMap {
        $0.score
    }
    .subscribe(onNext: {
        print($0)
    })
    .disposed(by: disposeBag)
    
student.onNext(laura)
    
laura.score.onNext(85)
    
student.onNext(charlotte)
    
laura.score.onNext(95)
    
charlotte.score.onNext(100)
```

输出：
```
80
85
90
95
100
```

5. `flatMapLatest`

`flatMapLatest` 运算符实际上是两个运算符的组合：`map` 和 `switchLatest`。`switchLatest` 将从最近的可观察量生成值，并从上一个可观察量中取消订阅。

```
let laura = Student(score: BehaviorSubject(value: 80))
let charlotte = Student(score: BehaviorSubject(value: 90))
    
let student = PublishSubject<Student>()
    
student
    .flatMapLatest { std in
        std.score
    }
    .subscribe(onNext: {
        print($0)
    })
    .disposed(by: disposeBag)
    
student.onNext(laura)
laura.score.onNext(85)
student.onNext(charlotte)
    
laura.score.onNext(95)
charlotte.score.onNext(100)
```

这里只有一件事要指出，它与之前的 `flatMap` 示例不同：
> 1. 在这里改变 laura 的分数不会有任何影响。它不会被打印出来。这是因为 flatMapLatest 切换到了最新的可观测值，Charlotte：

输出：
```
80
85
90
100
```

#### Observing events

有时，您可能希望将可观察量转换为其事件的可观察量。这很有用的一个典型方案是，当您无法控制具有可观察属性的可观察量，并且您希望处理错误事件以避免终止外部序列时。

6. `materialize` 和 `dematerialize`

```
enum MyError: Error {
    case anError
}
    
let laura = Student(score: BehaviorSubject(value: 80))
let charlotte = Student(score: BehaviorSubject(value: 100))
    
let student = BehaviorSubject(value: laura)
    
let studentScore = student
    .flatMapLatest {
        $0.score
    }
    
studentScore
    .subscribe(onNext: {
        print($0)
    })
    .disposed(by: disposeBag)
    
laura.score.onNext(85)
    
laura.score.onError(MyError.anError)
    
laura.score.onNext(90)
    
student.onNext(charlotte)
```

输出：
```
80
85
Unhandled error happened: anError
```

该例子中：

> 1. 使用`flatMapLatest`创建一个学生分数可观察序列，以访问学生的值属性
> 2. 订阅并在每个打印出每个分数
> 3. 将分数、错误和另一个分数添加到当前的学生身上
> 4. 将第二个学生 *charlotte* 添加到可观察的 `student` 中。由于使用了 `flatMapLatest`，因此将切换到此新学生并订阅她的分数

错误未处理。*studentScore* 可观察序列终止，外部学生可观察序列也终止了。

使用 `materialize` 运算符，可以将可观察序列发出的每个事件包装在可观察序列中。

``` 
let laura = Student(score: BehaviorSubject(value: 80))
let charlotte = Student(score: BehaviorSubject(value: 100))
    
let student = BehaviorSubject(value: laura)
    
let studentScore = student
    .flatMapLatest {
        $0.score.materialize()
    }
    
studentScore
    .subscribe(onNext: {
        print($0)
    })
    .disposed(by: disposeBag)
    
laura.score.onNext(85)
    
laura.score.onError(MyError.anError)
    
laura.score.onNext(90)
    
student.onNext(charlotte)
```

输出：
```
next(80)
next(85)
error(anError)
next(100)
```

现在处理的是事件，而不是元素。这就是 `dematerialize` 的用武之地。它将 `materialize` 的可观察序列转换回其原始形式。

```
let laura = Student(score: BehaviorSubject(value: 80))
let charlotte = Student(score: BehaviorSubject(value: 100))
    
let student = BehaviorSubject(value: laura)
    
let studentScore = student
    .flatMapLatest {
        $0.score.materialize()
    }
    
studentScore
    .filter {
        guard $0.error == nil else {
            print($0.error!)
            return false
        }

        return true
    }
    .dematerialize()
    .subscribe(onNext: {
        print($0)
    })
    .disposed(by: disposeBag)
    
laura.score.onNext(85)
    
laura.score.onError(MyError.anError)
    
laura.score.onNext(90)
    
student.onNext(charlotte)
```

输出：
```
80
85
anError
100
```

总结这个例子：

> 1. 打印并过滤掉任何错误。 
> 2. 使用 `dematerialize` 将可观察到的 studentScore 返回到其原始形式，发出分数和停止事件，而不是分数事件和停止事件的事件。