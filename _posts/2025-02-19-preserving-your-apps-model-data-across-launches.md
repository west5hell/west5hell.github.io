---
layout: post
title: "Preserving your app’s model data across launches"
date: 2025-02-19 18:00:00 +0800
categories: SwiftData
---

# 在应用多次启动之间保持模型数据

使用 SwiftData 提供的宏来定义模型类，并存储这些模型的实例，以确保它们在应用运行结束后仍然存在。

## Overview

大多数应用都会定义一些自定义类型来表示其创建或使用的数据。例如，旅行应用可能会定义代表行程、航班和预订住宿的类。借助 SwiftData，你可以快速高效地持久化这些数据，使其在应用多次启动之间保持可用，并利用框架与 SwiftUI 的集成，重新获取数据并将其显示在屏幕上。

SwiftData 旨在增强你现有的模型类。它提供了宏和属性包装器等工具，让你可以用 Swift 代码直观地描述应用的数据结构，而无需依赖外部资源，如模型文件或迁移映射文件。

### 将类转换为模型，使其可持久化存储。

要让 SwiftData 保存模型类的实例，首先需要导入该框架，并使用 [Model()](<https://developer.apple.com/documentation/swiftdata/model()>) 宏对类进行注解。该宏会使类符合 [PersistentModel](https://developer.apple.com/documentation/swiftdata/persistentmodel) 协议，SwiftData 通过该协议检查类并生成内部模式（schema）。此外，该宏还会让类符合 [Observable](https://developer.apple.com/documentation/Observation/Observable) 协议，从而实现对数据变更的跟踪。

```Swift
import SwiftData

// Annotate new or existing model classes with the @Model macro.
@Model
class Trip {
    var name: String
    var destination: String
    var startDate: Date
    var endDate: Date
    var accommodation: Accommodation?
}
```

默认情况下，SwiftData 会将类中所有非计算属性纳入模型，只要它们的类型是兼容的。该框架支持诸如 [Bool](https://developer.apple.com/documentation/Swift/Bool)、[Int](https://developer.apple.com/documentation/Swift/Int) 和 [String](https://developer.apple.com/documentation/Swift/String) 等基本类型，以及符合 [Codable](https://developer.apple.com/documentation/Swift/Codable) 协议的结构体、枚举等复杂值类型。

你编写的模型类代码现在成为应用模型层的唯一数据源，SwiftData 会利用它来确保持久化数据保持一致。

### 自定义模型属性的持久化行为

属性是 SwiftData 管理的模型类中的一个属性。通常情况下，框架对属性的默认处理方式已经足够。然而，如果你需要调整 SwiftData 持久化某个属性的方式，可以使用提供的模式宏（schema macros）。例如，为了避免模型数据冲突，你可以指定某个属性的值在所有实例中必须唯一。

要自定义属性的行为，可以使用 [Attribute(\_:originalName:hashModifier:)](<https://developer.apple.com/documentation/swiftdata/attribute(_:originalname:hashmodifier:)>) 宏对属性进行注解，并根据需求设置相应的选项来控制其行为：

```Swift
@Attribute(.unique) var name: String
```

除了强制唯一性约束外，@Attribute 还支持其他功能，例如保留已删除的值、Spotlight 索引和加密。此外，如果你希望在底层模型数据中保留原始名称，还可以使用 @Attribute 宏来正确处理属性重命名。

当一个模型包含一个属性，其类型也是一个模型（或模型的集合）时，SwiftData 会为你隐式管理这些模型之间的关系。默认情况下，在你删除一个相关的模型实例后，框架会将关系属性设置为 nil。要指定不同的删除规则，请使用 [Relationship(\_:deleteRule:minimumModelCount:maximumModelCount:originalName:inverse:hashModifier:)](<https://developer.apple.com/documentation/swiftdata/relationship(_:deleterule:minimummodelcount:maximummodelcount:originalname:inverse:hashmodifier:)>) 宏来注释该属性。例如，你可能希望在删除一个旅程时删除任何相关的住宿。有关删除规则的更多信息，请参阅 [Schema.Relationship.DeleteRule](https://developer.apple.com/documentation/swiftdata/schema/relationship/deleterule-swift.enum)。

```Swift
@Relationship(.cascade) var accommodation: Accommodation?
```

SwiftData 默认会持久化模型的所有非计算属性，但你可能并不总是希望这样做。例如，一个类上的一个或多个属性可能只包含不需要保存的临时数据，例如即将到来的旅程目的地的当前天气。在这种情况下，使用 [Transient()](<https://developer.apple.com/documentation/swiftdata/transient()>) 宏注释这些属性，SwiftData 将不会把它们的值写入磁盘。

```Swift
@Transient var destinationWeather = Weather.current()
```

### 配置模型存储

在 SwiftData 能够检查您的模型并生成所需的模式之前，您需要在运行时告诉它要持久化哪些模型，以及可选地，用于底层存储的配置。例如，您可能希望在运行测试时存储仅存在于内存中，或者在跨设备同步模型数据时使用特定的 CloudKit 容器。

要设置默认存储，请使用 [modelContainer(for:inMemory:isAutosaveEnabled:isUndoEnabled:onSetup:)](<https://developer.apple.com/documentation/SwiftUI/View/modelContainer(for:inMemory:isAutosaveEnabled:isUndoEnabled:onSetup:)-18hhy>) 视图修饰符（或场景等效项）并指定要持久化的模型类型数组。如果使用视图修饰符，请将其添加到视图层次结构的最顶层，以便所有嵌套视图继承正确配置的环境：

```Swift
import SwiftUI
import SwiftData


@main
struct TripsApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .modelContainer(for: [
                    Trip.self,
                    Accommodation.self
                ])
        }
    }
}
```

如果你没有使用 SwiftUI，请使用适当的初始化器手动创建一个模型容器：

```Swift
import SwiftData


let container = try ModelContainer([
    Trip.self,
    Accommodation.self
])
```

    Tip

    如果模型类型包含关系，则可以从数组中省略目标模型类型。SwiftData 会自动遍历模型的关系，并为您包含任何目标模型类型。

或者，使用 [ModelConfiguration](https://developer.apple.com/documentation/swiftdata/modelconfiguration) 来创建自定义存储。该类型提供了许多配置选项，包括：

- 存储是否仅存在于内存中。
- 存储是否为只读。
- 应用程序是否使用特定的 App Group 来存储其模型数据。

```Swift
let configuration = ModelConfiguration(isStoredInMemoryOnly: true, allowsSave: false)


let container = try ModelContainer(
    for: Trip.self, Accommodation.self,
    configurations: configuration
)
```

    Important

    自动 iCloud 同步依赖于 CloudKit 授权的存在，SwiftData 使用在该授权中找到的第一个容器。如果你的应用程序需要特定的容器，请使用 ModelConfiguration 的实例来指定该容器。

### 保存模型以供将来使用

要管理运行时模型类的实例，请使用模型上下文 — 该对象负责内存中的模型数据以及与模型容器的协调，以成功持久化这些数据。要获取绑定到主 actor 的模型容器的上下文，请使用 [modelContext](https://developer.apple.com/documentation/SwiftUI/EnvironmentValues/modelContext) 环境变量：

```Swift
import SwiftUI
import SwiftData


struct ContentView: View {
    @Environment(\.modelContext) private var context
}
```

在视图之外，或者如果你没有使用 SwiftUI，可以使用模型容器直接访问相同的绑定到 actor 的上下文：

```Swift
let context = container.mainContext
```

在这两种情况下，返回的上下文会定期检查其中是否包含未保存的更改，如果有，则会为您隐式保存这些更改。对于手动创建的上下文，请将 [autosaveEnabled](https://developer.apple.com/documentation/swiftdata/modelcontext/autosaveenabled) 属性设置为 true 以获得相同的行为。

要使 SwiftData 能够持久化一个模型实例并开始跟踪对它的更改，请将该实例插入到上下文中：

```Swift
var trip = Trip(name: name,
                destination: destination,
                startDate: startDate,
                endDate: endDate)


context.insert(trip)
```

插入之后，您可以通过调用上下文的 [save()](<https://developer.apple.com/documentation/swiftdata/modelcontext/save()>) 方法立即保存，或者依赖于上下文的隐式保存行为。上下文会自动跟踪其已知模型实例的更改，并在后续保存中包含这些更改。除了保存之外，您还可以使用上下文来获取、枚举和删除模型实例。有关更多信息，请参阅 [ModelContext](https://developer.apple.com/documentation/swiftdata/modelcontext)。

### 获取模型以进行显示或进一步处理

在您开始持久化模型数据之后，您可能希望检索该数据（具体化为模型实例），并在视图中显示这些实例或对它们执行其他操作。SwiftData 提供了 [Query](https://developer.apple.com/documentation/swiftdata/query) 属性包装器和 [FetchDescriptor](https://developer.apple.com/documentation/swiftdata/fetchdescriptor) 类型来执行获取操作。

要获取模型实例，并可选择性地应用搜索条件和首选排序顺序，请在您的 SwiftUI 视图中使用 @Query。@Model 宏为您的模型类添加了 Observable 一致性，使 SwiftUI 能够在任何获取的实例发生更改时刷新包含视图。

```Swift
import SwiftUI
import SwiftData


struct ContentView: View {
    @Query(sort: \.startDate, order: .reverse) var allTrips: [Trip]

    var body: some View {
        List {
            ForEach(allTrips) {
                TripView(for: $0)
            }
        }
    }
}
```

在视图之外，或者如果您没有使用 SwiftUI，请使用 [ModelContext](https://developer.apple.com/documentation/swiftdata/modelcontext) 上的两个获取方法之一。每个方法都需要一个 [FetchDescriptor](https://developer.apple.com/documentation/swiftdata/fetchdescriptor) 实例，其中包含一个谓词和一个排序顺序。获取描述符允许进行额外的配置，这些配置会影响批处理、偏移和预取等。

```Swift
let context = container.mainContext


let upcomingTrips = FetchDescriptor<Trip>(
    predicate: #Predicate { $0.startDate > Date.now },
    sortBy: [
        .init(\.startDate)
    ]
)
upcomingTrips.fetchLimit = 50
upcomingTrips.includePendingChanges = true


let results = context.fetch(upcomingTrips)
```

有关谓词的更多信息，请参阅 [Predicate](https://developer.apple.com/documentation/foundation/predicate)。
