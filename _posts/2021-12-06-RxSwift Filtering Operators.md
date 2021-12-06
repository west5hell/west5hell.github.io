---
layout: post
title:  RxSwift Filtering Operators（筛选操作符）
date:   2021-12-6
---

### Ignoring operators（忽略操作符）
1. `ignoreElements()` 忽略所有的 next 事件。然而，允许终止时间通过，例如 completed 或者 error 事件。


```
    let strikes = PublishSubject<String>()
    
    let disposeBag = DisposeBag()
    
    strikes
        .ignoreElements()
        .subscribe { _ in
            print("You're out")
        }
        .disposed(by: disposeBag)
    
    strikes.onNext("X")
    strikes.onNext("Y")
    strikes.onNext("Z")
    
    strikes.onCompleted()
```

如果将 `.ignoreElements()` 注释掉，会输出：
```
You're out
You're out
You're out
You're out
```
三个 next 事件加一个完成事件。

取消注释：
```
You're out
```
只输出完成事件，其他 next 事件都被忽略掉。

2. `element(at index: Int)`

忽略除输入的索引以外的事件（不包括终止事件）
```
    let strikes = PublishSubject<String>()
    
    let disposeBag = DisposeBag()
    
    strikes
        .element(at: 1)
        .subscribe { e in
            print(e)
        }
        .disposed(by: disposeBag)
    
    strikes.onNext("X")
    strikes.onNext("Y")
    strikes.onNext("Z")
    
    strikes.onCompleted()
```

输出：
```
next(Y)
completed
```
一旦有元素在提供的索引处被发射出来，订阅就会终止。

3. `filter` 需要一个闭包作为过滤的条件

```
    let disposeBag = DisposeBag()
    
    Observable.of(1, 2, 3, 4, 5, 6)
        .filter { element in
            element.isMultiple(of: 2)
        }
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
```

输出：
```
2
4
6
```

4. `skip` 跳过操作符

```
    let disposeBag = DisposeBag()
    
    Observable.of("A", "B", "C", "D", "E", "F")
        .skip(3)
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
```

输出：
```
D
E
F
```

5. `skip(while: )` 跳过满足条件的元素，直到不满足条件，然后输出元素

```
    let disposeBag = DisposeBag()
    
    Observable.of(2, 2, 3, 4, 5)
        .skip(while: { $0.isMultiple(of: 2) })
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
```

输出：
```
3
4
5
```

6. `skip(until other: ObervableType)` 

持续跳过源可观察序列，直到其他观察序列发射事件（非终止事件）出来
```
    let disposeBag = DisposeBag()
    
    let subject = PublishSubject<String>()
    let trigger = PublishSubject<String>()
    
    subject
        .skip(until: trigger)
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
    
    subject.onNext("A")
    subject.onNext("B")
    trigger.onNext("X")
    subject.onNext("C")
```
输出：
```
C
```

7. `take(_ count: Int)` 与 `skip` 相反，忽略`count`之后的元素

```
    let disposeBag = DisposeBag()
    
    Observable.of(1, 2, 3, 4, 5, 6)
        .take(3)
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
```

8. `take(while: Closure)` 与 `skip(while: )` 类似，但效果相反

```
    let disposeBag = DisposeBag()
    
    Observable.of(1, 11, 2, 32, 4, 33, 46)
        .take(while: { $0 < 30 })
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
```
输出：
```
1
11
2
```
如果想使用发射的元素的索引，可以使用枚举操作符`enumerated`
```
    let disposeBag = DisposeBag()
    Observable.of(2, 2, 4, 4, 6, 6)
        .enumerated()
        .take(while: { index, integer in
            integer.isMultiple(of: 2) && index < 3
        })
        .map(\.element)
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
```
输出：
```
2
2
4
```

9. `take(until: , behavior: TakeBehavior = .exclusive)` 获取元素直到满足预测条件

```
    let disposeBag = DisposeBag()
    
    Observable.of(1, 11, 2, 32, 4, 33, 46)
        .take(until: { $0.isMultiple(of: 4) }, behavior: .inclusive)
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
```

输出：
```
1
11
2
32
```
`exclusive` 输出：
```
1
11
2
```

10. `take(until other: Source` 和`skip(until other: ObervableType)` 类似

```
let disposeBag = DisposeBag()
    
    let subject = PublishSubject<String>()
    let trigger = PublishSubject<String>()
    
    subject
        .take(until: trigger)
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
    
    subject.onNext("1")
    subject.onNext("2")
    
    trigger.onNext("X")
    subject.onNext("3")
```
输出：
```
1
2
```

11. `distinctUntilChanged` 去除紧挨着的重复项

```
let disposeBag = DisposeBag()
    
    Observable.of("A", "A", "B", "B", "A")
        .distinctUntilChanged()
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
```
输出：
```
A
B
A
```

12. `distinctUntilChanged(_:)`

如果事件传递的元素符合 Equatable，可以选择使用 `distinctUntilChanged(_:)`来提供你自己的自定义逻辑测试相等，传递的参数是一个比较器。

```
    let disposeBag = DisposeBag()
    
    let formatter = NumberFormatter()
    formatter.numberStyle = .spellOut
    
    Observable<NSNumber>.of(10, 110, 20, 200, 210, 310)
        .distinctUntilChanged { a, b in
            guard let aWords = formatter.string(from: a)?.components(separatedBy: " "),
                  let bWords = formatter.string(from: b)?.components(separatedBy: " ") else {
                return false
            }
            
            var containMatch = false
            for aWord in aWords where bWords.contains(aWord) {
                containMatch = true
                break
            }
            return containMatch
        }
        .subscribe(onNext: {
            print($0)
        })
        .disposed(by: disposeBag)
```
输出：
```
10
20
200
```