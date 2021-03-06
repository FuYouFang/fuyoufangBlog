# Swift ReactorKit 框架

![ReactorKit](../images/ReactorKit_logo.png)

[ReactorKit](https://github.com/ReactorKit/ReactorKit) 是一个响应式、单向 Swift 应用框架。下面来介绍一下 ReactorKit 当中的基本概念和使用方法。

## 目录

* [基本概念](#基本概念)
    * [设计目标](#设计目标)
    * [View](#view)
    * [Reactor](#Reactor-反应堆)
* [高级用法](#高级用法)
    * [Global States (全局状态) ](#Global-States-(全局状态) )
    * [View Communication (View 通信)](#View-Communication-(View-通信))
    * [Testing 测试](#Testing-测试)
    * [Scheduling 调度](#Scheduling-调度)
* [示例](#示例)
* [依赖](#依赖)
* [其他](#其他)

## 基本概念

ReactorKit 是 [Flux](https://facebook.github.io/flux/) 和 [Reactive Programming](https://en.wikipedia.org/wiki/Reactive_programming) 的混合体。用户的操作和视图 view 的状态通过可被观察的流传递到各层。这些流是单向的：视图 view 仅能发出操作（action）流 ，反应堆仅能发出状态（states）流。

<p align="center">
  <img alt="flow" src="https://cloud.githubusercontent.com/assets/931655/25073432/a91c1688-2321-11e7-8f04-bf91031a09dd.png" width="600">
</p>

### 设计目标

* **可测性**：ReactorKit 的首要目标是将业务逻辑从视图 view 上分离。这可以让代码方便测试。一个反应堆不依赖于任何 view。这样就只需要测试反应堆和 view 数据的绑定。测试方法可点击[查看](#Testing-测试)。
* **侵入小**：ReactorKit 不要求整个应用采用这一种框架。对于一些特殊的 view，可以部分的采用 ReactorKit。对于现存的项目，不需要重写任何东西，就可以直接使用 ReactorKit。
* **更少的键入**：对于一些简单的功能，ReactorKit 可以减少代码的复杂度。和其他的框架相比，ReactorKit 需要的代码更少。可以从一个简单的功能开始，逐渐扩大使用的范围。


### View

*View* 用来展示数据。 view controller 和 cell 都可以看做一个 view。view 需要做两件事：（1）绑定用户输入的操作流，（2）将状态流绑定到 view 对应的 UI 元素。view 层没有业务逻辑，只负责绑定操作流和状态流。

定义一个 view，只需要将一个现存的类符合协议 `View`。然后这个类就自动有了一个 `reactor` 的属性。view 的这个属性通常由外界设置。

```swift
class ProfileViewController: UIViewController, View {
  var disposeBag = DisposeBag()
}

profileViewController.reactor = UserViewReactor() // inject reactor
```

当这个 `reactor` 属性被设置（或修改）的时候，将自动调用 `bind(reactor:)` 方法。view 通过实现 `bind(reactor:)` 来绑定操作流和状态流。

```swift
func bind(reactor: ProfileViewReactor) {
  // action (View -> Reactor)
  refreshButton.rx.tap.map { Reactor.Action.refresh }
    .bind(to: reactor.action)
    .disposed(by: self.disposeBag)

  // state (Reactor -> View)
  reactor.state.map { $0.isFollowing }
    .bind(to: followButton.rx.isSelected)
    .disposed(by: self.disposeBag)
}
```

#### Storyboard 的支持

如果使用 storyboard 来初始一个 view controller，则需要使用 `StoryboardView` 协议。`StoryboardView` 协议和 `View` 协议相比，唯一不同的是 `StoryboardView` 协议是在 view 加载结束之后进行绑定的。

```swift
let viewController = MyViewController()
viewController.reactor = MyViewReactor() // will not executes `bind(reactor:)` immediately

class MyViewController: UIViewController, StoryboardView {
  func bind(reactor: MyViewReactor) {
    // this is called after the view is loaded (viewDidLoad)
  }
}
```

### Reactor 反应堆

反应堆 *Reactor* 层，和 UI 无关，它控制着一个 view 的状态。reactor 最主要的作用就是将操作流从 view 中分离。每个 view 都有它对应的反应堆 reactor，并且将它所有的逻辑委托给它的反应堆 reactor。

定义一个 reactor 时需要符合 `Reactor` 协议。这个协议要求定义三个类型： `Action`, `Mutation` 和 `State`，另外它需要定义一个名为 `initialState` 的属性。

```swift
class ProfileViewReactor: Reactor {
  // represent user actions
  enum Action {
    case refreshFollowingStatus(Int)
    case follow(Int)
  }

  // represent state changes
  enum Mutation {
    case setFollowing(Bool)
  }

  // represents the current view state
  struct State {
    var isFollowing: Bool = false
  }

  let initialState: State = State()
}
```

`Action` 表示用户操作，`State` 表示 view 的状态，`Mutation` 是 `Action` 和 `State` 之间的转化桥梁。reactor 将一个 action 流转化到 state 流，需要两步：`mutate()` 和 `reduce()`。

<p align="center">
  <img alt="flow-reactor" src="https://cloud.githubusercontent.com/assets/931655/25098066/2de21a28-23e2-11e7-8a41-d33d199dd951.png" width="800">
</p>

#### `mutate()`

`mutate()` 接受一个 `Action`，然后产生一个 `Observable<Mutation>`。

```swift
func mutate(action: Action) -> Observable<Mutation>
```

所有的副作用应该在这个方法内执行，比如异步操作，或者 API 的调用。

```swift
func mutate(action: Action) -> Observable<Mutation> {
  switch action {
  case let .refreshFollowingStatus(userID): // receive an action
    return UserAPI.isFollowing(userID) // create an API stream
      .map { (isFollowing: Bool) -> Mutation in
        return Mutation.setFollowing(isFollowing) // convert to Mutation stream
      }

  case let .follow(userID):
    return UserAPI.follow()
      .map { _ -> Mutation in
        return Mutation.setFollowing(true)
      }
  }
}
```

#### `reduce()`

`reduce()` 由当前的 `State` 和一个 `Mutation` 生成一个新的 `State`。

```swift
func reduce(state: State, mutation: Mutation) -> State
```

这个应该是一个简单的方法。它应该仅仅同步的返回一个新的 `State`。不要在这个方法内执行任何有副作用的操作。

```swift
func reduce(state: State, mutation: Mutation) -> State {
  var state = state // create a copy of the old state
  switch mutation {
  case let .setFollowing(isFollowing):
    state.isFollowing = isFollowing // manipulate the state, creating a new state
    return state // return the new state
  }
}
```

#### `transform()`

`transform()` 用来转化每一种流。这里包含三种 `transforms()` 的方法。

```swift
func transform(action: Observable<Action>) -> Observable<Action>
func transform(mutation: Observable<Mutation>) -> Observable<Mutation>
func transform(state: Observable<State>) -> Observable<State>
```

通过这些方法可以将流进行转化，或者将流和其他流进行合并。例如：在合并全局事件流时，最好使用 `transform(mutation:)` 方法。点击查看[全局状态](#Global-States-(全局状态))的更多信息。

另外，也可以通过这些方法进行测试。

```swift
func transform(action: Observable<Action>) -> Observable<Action> {
  return action.debug("action") // Use RxSwift's debug() operator
}
```

## 高级用法

### Global States (全局状态) 

和 Redux 不同, ReactorKit 不需要一个全局的 app state，这意味着你可以使用任何类型来管理全局 state，例如用 `BehaviorSubject`，或者 `PublishSubject`，甚至一个 reactor。ReactorKit 不需要一个全局状态，所以不管应用程序有多特殊，都可以使用 ReactorKit。

在 **Action → Mutation → State** 流中，没有使用任何全局的状态。你可以使用 `transform(mutation:)` 将一个全局的 state 转化为 mutation。例如：我们使用一个全局的 `BehaviorSubject` 来存储当前授权的用户，当 `currentUser` 变化时，需要发出 `Mutation.setUser(User?)`，则可以采用下面的方案：

```swift
var currentUser: BehaviorSubject<User> // global state

func transform(mutation: Observable<Mutation>) -> Observable<Mutation> {
    return Observable.merge(mutation, currentUser.map(Mutation.setUser))
}
```
这样，当 view 每次向 reactor 产生一个 action 或者 `currentUser` 改变的时候，都会发送一个 mutation。

### View Communication (View 通信)

多个 view 之间通信时，通常会采用回调闭包或者代理模式。ReactorKit 建议采用 [reactive extensions](https://github.com/ReactiveX/RxSwift/blob/master/RxSwift/Reactive.swift) 来解决。最常见的 `ControlEvent` 示例是 `UIButton.rx.tap`。关键思路就是将自定义的视图转化为像 UIButton 或者 UILabel 一样。

<p align="center">
  <img alt="view-view" src="https://user-images.githubusercontent.com/931655/27789114-393e2eea-6026-11e7-9b32-bae314e672ee.png" width="600">
</p>

假设我们有一个 `ChatViewController` 来展示消息。 `ChatViewController` 有一个 `MessageInputView`，当用户点击 `MessageInputView` 上的发送按钮时，文字将会发送到 `ChatViewController`，然后 `ChatViewController` 绑定到对应的 reactor 的 action。下面是 `MessageInputView` 的 reactive extensions 的一个示例：

```swift
extension Reactive where Base: MessageInputView {
    var sendButtonTap: ControlEvent<String> {
        let source = base.sendButton.rx.tap.withLatestFrom(...)
        return ControlEvent(events: source)
    }
}
```

这样就是可以在 `ChatViewController` 中使用这个扩展。例如：

```swift
messageInputView.rx.sendButtonTap
  .map(Reactor.Action.send)
  .bind(to: reactor.action)
```

### Testing 测试

ReactorKit 有一个用于测试的 built-in 功能。通过下面的指导，你可以很容易测试 view 和 reactor。

#### 测试内容

首先，你要确定测试内容。有两个方面需要测试，一个是 view 或者一个是 reactor。

* View
    * Action: 能否通过给定的用户交互发送给 reactor 对应的 action？
    * State: view 能否根据给定的 state 对属性进行正确的设置？
* Reactor
    * State: state 能否根据 action 进行相应的修改？

#### View 测试

view 可以根据 *stub* reactor 进行测试。reactor 有一个 `stub` 的属性，它可以打印 actions，并且强制修改 states。如果启用了 reactor 的 stub，`mutate()` 和 `reduce()` 将不会被执行。stub 有下面几个属性：

```swift
var isEnabled: Bool { get set }
var state: StateRelay<Reactor.State> { get }
var action: ActionSubject<Reactor.Action> { get }
var actions: [Reactor.Action] { get } // recorded actions
```

下面是一些测试示例：

```swift
func testAction_refresh() {
  // 1. prepare a stub reactor
  let reactor = MyReactor()
  reactor.stub.isEnabled = true

  // 2. prepare a view with a stub reactor
  let view = MyView()
  view.reactor = reactor

  // 3. send an user interaction programatically
  view.refreshControl.sendActions(for: .valueChanged)

  // 4. assert actions
  XCTAssertEqual(reactor.stub.actions.last, .refresh)
}

func testState_isLoading() {
  // 1. prepare a stub reactor
  let reactor = MyReactor()
  reactor.stub.isEnabled = true

  // 2. prepare a view with a stub reactor
  let view = MyView()
  view.reactor = reactor

  // 3. set a stub state
  reactor.stub.state.value = MyReactor.State(isLoading: true)

  // 4. assert view properties
  XCTAssertEqual(view.activityIndicator.isAnimating, true)
}
```

#### 测试 Reactor 

reactor 可以被单独测试。

```swift
func testIsBookmarked() {
    let reactor = MyReactor()
    reactor.action.onNext(.toggleBookmarked)
    XCTAssertEqual(reactor.currentState.isBookmarked, true)
    reactor.action.onNext(.toggleBookmarked)
    XCTAssertEqual(reactor.currentState.isBookmarked, false)
}
```

一个 action 有时会导致 state 多次改变。比如，一个 `.refresh` action 首先将 `state.isLoading` 设置为 `true`，并在刷新结束后设置为 `false`。在这种情况下，很难用 `currentState` 测试 `state` 的 `isLoading` 的状态更改过程。这时，你可以使用 [RxTest](https://github.com/ReactiveX/RxSwift) 或 [RxExpect](https://github.com/devxoul/RxExpect)。下面是使用 RxExpect 的测试案例：

```swift
func testIsLoading() {
  RxExpect("it should change isLoading") { test in
    let reactor = test.retain(MyReactor())
    test.input(reactor.action, [
      next(100, .refresh) // send .refresh at 100 scheduler time
    ])
    test.assert(reactor.state.map { $0.isLoading })
      .since(100) // values since 100 scheduler time
      .assert([
        true,  // just after .refresh
        false, // after refreshing
      ])
  }
}
```

### Scheduling 调度

定义 `scheduler` 属性来指定发出和观察的状态流的 `scheduler`。注意：这个队列 **必须** 是一个串行队列。`scheduler` 的默认值是 `CurrentThreadScheduler`。

```swift
final class MyReactor: Reactor {
  let scheduler: Scheduler = SerialDispatchQueueScheduler(qos: .default)

  func reduce(state: State, mutation: Mutation) -> State {
    // executed in a background thread
    heavyAndImportantCalculation()
    return state
  }
}
```

## 示例

* [Counter](https://github.com/ReactorKit/ReactorKit/tree/master/Examples/Counter): The most simple and basic example of ReactorKit
* [GitHub Search](https://github.com/ReactorKit/ReactorKit/tree/master/Examples/GitHubSearch): A simple application which provides a GitHub repository search
* [RxTodo](https://github.com/devxoul/RxTodo): iOS Todo Application using ReactorKit
* [Cleverbot](https://github.com/devxoul/Cleverbot): iOS Messaging Application using Cleverbot and ReactorKit
* [Drrrible](https://github.com/devxoul/Drrrible): Dribbble for iOS using ReactorKit ([App Store](https://itunes.apple.com/us/app/drrrible/id1229592223?mt=8))
* [Passcode](https://github.com/cruisediary/Passcode): Passcode for iOS RxSwift, ReactorKit and IGListKit example
* [Flickr Search](https://github.com/TaeJoongYoon/FlickrSearch): A simple application which provides a Flickr Photo search with RxSwift and ReactorKit
* [ReactorKitExample](https://github.com/gre4ixin/ReactorKitExample)

## 依赖

* [RxSwift](https://github.com/ReactiveX/RxSwift) >= 5.0

## 其他

其他信息可以查看 [github](https://github.com/ReactorKit/ReactorKit/)
