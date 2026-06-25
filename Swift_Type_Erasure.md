# Swift 中的类型擦除 (Type Erasure) 技术

## 1. 为什么需要类型擦除？

在 Swift 中，Protocol 是非常强大的，但一旦 Protocol 拥有了 **Associated Type (关联类型)** 或者使用了 `Self` 约束（统称为 **PATs**, Protocols with Associated Types），它就不能再像普通 Protocol 那样直接作为变量类型、函数参数或数组元素使用了。

### 痛点示例
假设我们有一个简单的协议：
```swift
protocol Animal {
    associatedtype Food
    func eat(_ food: Food)
}

struct Cat: Animal {
    func eat(_ food: String) { print("Cat eating \(food)") }
}

struct Dog: Animal {
    func eat(_ food: String) { print("Dog eating \(food)") }
}

// ❌ 编译错误：Protocol 'Animal' can only be used as a generic constraint because it has Self or associated type requirements
// let animals: [Animal] = [Cat(), Dog()] 
```
编译器无法确定 `Animal` 到底是什么大小，也不知道 `Food` 具体是什么类型，因此无法将它们放入同一个异构容器中。

**类型擦除**的目的就是为了解决这个问题：**将具体的类型信息“擦除”，包装成一个统一的类型，以便存储和传递。**

---

## 2. 经典的类型擦除技术 (Swift 5.7 之前)

在 Swift 5.7 之前，我们通常需要手动编写包装器（Wrapper）来实现类型擦除。

### 2.1 基于类的擦除 (Abstract Base Class 风格)
虽然 Swift 提倡值类型，但有时为了多态，我们会创建一个抽象基类（虽然 Swift 没有真正的抽象类，但可以通过 `fatalError` 模拟）。

### 2.2 基于 Box 的擦除 (AnyWrapper)
这是标准库中 `AnyHashable`, `AnySequence`, `AnyView` (SwiftUI) 采用的模式。

**实现原理**：创建一个泛型结构体，内部持有一个私有的、抽象的基类或闭包，将具体类型的操作转发给内部持有者。

```swift
// 1. 定义一个包装器
struct AnyAnimal<Food>: Animal {
    // 2. 内部私有属性，持有具体的实现
    private let _eat: (Food) -> Void
    
    // 3. 初始化器，接收任意符合协议的具体类型
    init<A: Animal>(_ animal: A) where A.Food == Food {
        // 将方法调用通过闭包捕获
        self._eat = animal.eat
    }
    
    // 4. 实现协议方法，转发调用
    func eat(_ food: Food) {
        _eat(food)
    }
}

// ✅ 现在可以使用了
let cat = Cat()
let dog = Dog()
let animals: [AnyAnimal<String>] = [AnyAnimal(cat), AnyAnimal(dog)]
```
这种方式非常繁琐，需要编写大量的样板代码。

---

## 3. 现代 Swift 的类型擦除 (Swift 5.7+)

Swift 5.7 引入了巨大的改进，极大地简化了类型擦除的需求。

### 3.1 Opaque Types (`some Protocol`)
`some` 关键字表示“某种符合该协议的具体类型”，但对外隐藏了具体是谁。它保证了类型的一致性（即编译器知道它是固定的某一种类型），主要用于返回值。

```swift
func makeAnimal() -> some Animal {
    return Cat() // 编译器知道这是 Cat，但调用者只知道是 some Animal
}
```
注意：`some` 并没有完全解决异构数组的问题，因为 `some Animal` 仍然是一个特定的（虽然不透明的）类型。

### 3.2 Existential Types (`any Protocol`)
`any` 关键字创建了一个**存在类型 (Existential Type)**。这才是真正的、编译器自动生成的“类型擦除盒子”。

从 Swift 5.7 开始，你可以直接使用 `any Animal`，编译器会在幕后为你处理包装工作。

```swift
// ✅ Swift 5.7+ 直接支持！
let animals: [any Animal] = [Cat(), Dog()] 

for animal in animals {
    // 注意：这里使用 animal 时受到限制，因为编译器不知道具体的 Food 是什么
    // animal.eat("Fish") // ❌ 仍然可能报错，因为编译器不知道这个 animal 接受 String
}
```

### 3.3 Primary Associated Types (主要关联类型)
为了解决 `any Protocol` 中关联类型丢失的问题，Swift 引入了 `<Type>` 语法来约束关联类型。

```swift
// 修改协议，指定 Food 为主要关联类型
protocol Animal<Food> {
    associatedtype Food
    func eat(_ food: Food)
}

// ✅ 现在可以指定关联类型了
let animals: [any Animal<String>] = [Cat(), Dog()]

for animal in animals {
    animal.eat("Fish") // ✅ 编译器现在知道 Food 是 String，可以安全调用！
}
```

---

## 4. 总结与建议

| 场景 | 推荐方案 | 说明 |
| :--- | :--- | :--- |
| **函数返回值** | `some Protocol` | 性能最好，保留了静态类型信息，允许编译器优化。 |
| **函数参数** | `some Protocol` | 等价于泛型 `func foo<T: P>(t: T)`，但在语法上更简洁。 |
| **异构集合 (Array/Dict)** | `any Protocol` | 当你需要在一个数组里放不同类型的对象时使用。 |
| **保存为属性** | `any Protocol` | 当你需要存储一个依赖，且不希望将泛型参数传染给整个类型时。 |

**最佳实践**：
1.  **优先使用 `some`**：默认情况下，尝试使用 `some`。它保留了完整的类型信息，性能开销最小。
2.  **需要动态性时用 `any`**：当你确实需要存储不同类型的对象，或者需要动态改变变量持有的具体类型时，使用 `any`。
3.  **忘掉手动编写 `AnyWrapper`**：除非你需要支持非常老的 Swift 版本，否则现代 Swift (`any` + Primary Associated Types) 已经覆盖了绝大多数需求。
