# 协议中的私有属性
在 Swift 中，协议不能对其申明的属性进行指定的访问控制。如果协议中列出了某个属性，则必须使遵守协议的类型显式声明这些属性。  

但有些时候，虽然你需要这些属性对现实提供支持，但是你并不希望这些属性在类型的外部被使用。让我们看看如何解决这个问题。  

## 一个简单的例子
假设你需要创建一个专门的对象来管理你的 ViewControllers ( 视图控制器 ) 导航，就像一个 Coordinator ( 协调器 )。  

每个协调器都有一个根 `UINavigationController`，并共享一些通用的功能，比如在它上面推进和弹出其他 ViewController。所以最初它看起来可能是这样 <sup>1</sup> ：  

```
// Coordinator.swift

protocol Coordinator {
  var navigationController: UINavigationController { get }
  var childCoordinator: Coordinator? { get set }

  func push(viewController: UIViewController, animated: Bool)
  func present(childViewController: UIViewController, animated: Bool)
  func pop(animated: Bool)
}

extension Coordinator {
  func push(viewController: UIViewController, animated: Bool = true) {
    self.navigationController.pushViewController(viewController, animated: animated)
  }
  func present(childCoordinator: Coordinator, animated: Bool) {
    self.navigationController.present(childCoordinator.navigationController, animated: animated) { [weak self] in
      self?.childCoordinator = childCoordinator
    }
  }
  func pop(animated: Bool = true) {
    if let childCoordinator = self.childCoordinator {
      self.dismissViewController(animated: animated) { [weak self] in
        self?.childCoordinator = nil
      }
    } else {
      self.navigationController.popViewController(animated: animated)
    }
  }
}
```

当我们想要声明一个新的 `Coordinator` 对象时，我们会像这样做：  
```
// MainCoordinator.swift

class MainCoordinator: Coordinator {
  let navigationController: UINavigationController = UINavigationController()
  var childCoordinator: Coordinator?

  func showTutorialPage1() {
    let vc = makeTutorialPage(1, coordinator: self)
    self.push(viewController: vc)
  }
  func showTutorialPage2() {
    let vc = makeTutorialPage(2, coordinator: self)
    self.push(viewController: vc)
  }

  private func makeTutorialPage(_ num: Int, coordinator: Coordinator) -> UIViewController { … }
}
```

## 问题：泄漏实现细节
这个解决方案在 `protocol` 的可见性上有两个问题：  

- 每当我们想要申明一个新的 `Coordinator` 对象，我们都必须显式的声明一个 `let navigationController: UINavigationController` 属性和一个 `var childCoordinator: Coordinator?` 属性。**甚至，当我们在遵守协议的类型的现实中并不需要显式的使用他们时** - 它们就在那里，因为我们需要它们作为默认的实现来供 `protocol Coordinator` 正常工作。  

- 我们必须声明的这两个属性需要具有与我们的 `MainCoordinator` 相同的可见性 ( 在本例中为隐式 `internal (内部)` 访问控制级别 )，因为这是 `protocol Coordinator` 的必须条件。这使得它们对外部可见，比如使用 `MainCoordinator` 进行编码。  

所以问题是我们每次都要声明一些属性即使它只是一些实现细节，而且这些实现细节会通过外部接口被泄漏，从而允许类的访问者做一些本不应该被允许的事，例如：  

```
let mainCoord = MainCoordinator()
// 访问者不应该被允许直接访问 navigationController ，但是它们可以
mainCoord.navigationController.dismissViewController(animated: true)
// 他们也不应该被允许做这样的事情
mainCoord.childCoordinator = mainCoord
```

也许你会认为，既然我们不希望它们是可见的，那么可以直接在第一段代码的 `protocol` 中不声明这两个属性。但是如果我们这样做，我们将无法通过 `extension Coordinator` 来提供默认的实现，因为默认的实现需要这两个属性存在以便它们的代码被编译。  

你可能希望 Swift 允许在协议中申明这些属性为 `fileprivate`，但是在 Swift 4 中，你不能在 `protocols` 中使用任何访问控制的关键字。  

所以我们如何才能解决它，在给需要这些属性的默认实现提供支持的同时，又能防止它们通过外部接口被泄漏？

## 一个解决方案
实现这一点的一个技巧是将这些属性隐藏在中间对象中，并在该对象中将对应的属性申明为 `fileprivate`。  

通过这种方式，尽管我们依旧在对应类型的公共接口中声明了属性，但是接口的访问者却不能控制该对象的内部属性。同时我们对于协议的默认实现却能够访问它们 - 只要它们在同一个文件中被声明 ( 因为它们是 `fileprivate` )。  

看起来就像这样：  

```
// Coordinator.swift

class CoordinatorComponents {
  fileprivate let navigationController: UINavigationController = UINavigationController()
  fileprivate var childCoordinator: Coordinator? = nil
}

protocol Coordinator: AnyObject {
  var coordinatorComponents: CoordinatorComponents { get }

  func push(viewController: UIViewController, animated: Bool)
  func present(childCoordinator: Coordinator, animated: Bool)
  func pop(animated: Bool)
}

extension Coordinator {
  func push(viewController: UIViewController, animated: Bool = true) {
    self.coordinatorComponents.navigationController.pushViewController(viewController, animated: animated)
  }
  func present(childCoordinator: Coordinator, animated: Bool = true) {
    let childVC = childCoordinator.coordinatorComponents.navigationController
    self.coordinatorComponents.navigationController.present(childVC, animated: animated) { [weak self] in
      self?.coordinatorComponents.childCoordinator = childCoordinator // retain the child strongly
    }
  }
  func pop(animated: Bool = true) {
    let privateAPI = self.coordinatorComponents
    if privateAPI.childCoordinator != nil {
      privateAPI.navigationController.dismiss(animated: animated) { [weak privateAPI] in
        privateAPI?.childCoordinator = nil
      }
    } else {
      privateAPI.navigationController.popViewController(animated: animated)
    }
  }
}
```  

现在，遵守协议的 `MainCoordinator` 类型：  

- 仅仅需要申明一个单独的 `let coordinatorComponents = CoordinatorComponents() ` 属性，并不需要知道 `CoordinatorComponents` 类型的内部有些什么 ( 隐藏了实现细节 )  

- 在 `MainCoordinator.swift` 文件中，不能访问 `coordinatorComponents` 的任何属性，因为它们被申明为 `fileprivate`。  

```
public class MainCoordinator: Coordinator {
  let coordinatorComponents = CoordinatorComponents()

  func showTutorialPage1() {
    let vc = makeTutorialPage(1, coordinator: self)
    self.push(viewController: vc)
  }
  func showTutorialPage2() {
    let vc = makeTutorialPage(2, coordinator: self)
    self.push(viewController: vc)
  }

  private func makeTutorialPage(_ num: Int, coordinator: Coordinator) -> UIViewController { … }
}
```

当然，你仍然需要在遵守协议的类型中声明 `let coordinatorComponents` 来提供存储，这个声明必须是可见的 ( 不能是 `private` )，因为这是遵守 `protocol Coordinator` 所要求的一部分。但是：  

- 只需要声明 1 个属性，取代之前的 2 个 ( 在更复杂的情况下会有更多 )  

- 更重要的是，即使它可以从遵守协议的类型的实现中访问，也可以从外部接口访问，你却不能对它做任何事情。  

然当，你仍然可以访问 `myMainCoordinator.coordinatorComponents`，但是你不能使用它做任何事情，因为它所有的属性都是 `fileprivate` ！  

## 结论
Swift 可能无法提供你想要的所有功能。你可能希望有朝一日 `protocols` 允许对它声明需要的属性和方法使用访问控制关键字，或者通过某种方式将它们在公共 API 中隐藏。  

但与此同时，掌握这些技巧和变通方法可以使你的公共 API 更好、更安全，避免泄露实现细节或者访问在实现之外不应该被修改的属性，同时仍然使用 [Mixin pattern](http://alisoftware.github.io/swift/protocol/2015/11/08/mixins-over-inheritance/) 并提供默认实现。  

***
> 1. 这是一个简化的例子；不要将注意力集中在 Coordinator 的实现 - 它不是这个例子的重点，更应该关注的是需要在协议中声明公开可访问的属性。























