# ReactiveKit

__ReactiveKit__ is a collection of Swift frameworks for reactive and functional reactive programming.

* [ReactiveKit](https://github.com/ReactiveKit/ReactiveKit) - A Swift Reactive Programming Kit.
* [ReactiveFoundation](https://github.com/ReactiveKit/ReactiveFoundation) - NSFoundation extensions like type-safe KVO.
* [ReactiveUIKit](https://github.com/ReactiveKit/ReactiveUIKit) - UIKit extensions that enable bindings.

## Observable

`Observable` type represents observable mutable state, like a variable whose changes can be observed. To provide Observable, just initialize it with the initial value. To change its value afterwards, update `value` property.

```swift
let name = Observable("Jim")
name.value = "Jim Kirk"
```

To observe changes, register an observer with `observe` method.

```swift
name.observe { value in
  print(value)
}
```

You can also bind Observables to other Observables, like the ones on UIKit objects provided by ReactiveUIKit framework:

```swift
name.bindTo(nameLabel)
```

## ObservableCollection

`ObservableCollection` is a type similar to Observable, but used to encapsulate a collection (array, dictionary or set) in order to observe fine-grained changes done to the collection iself. Events generated by ObservableCollection contain both the new state of the collection (the collection itself) plus the information about what elements were inserted, updated or deleted. 

```swift
let names: ObservableCollection(["Steve", "Tim"])

names.observe { e in
  print("array: \(e.collection), inserts: \(e.inserts), updates: \(e.updates), deletes: \(e.deletes)")
}
```

You work with the ObservableCollection like you'd work with the collection it encapsulates.

```swift
names.append("John") // prints: array ["Steve", "Tim", "John"], inserts: [2], updates: [], deletes: []
names.removeLast()   // prints: array ["Steve", "Tim"], inserts: [], updates: [], deletes: [2]
names[1] = "Mark"    // prints: array ["Steve", "Mark"], inserts: [], updates: [1], deletes: []
```

ObservableCollection can be mapped, filtered and sorted. For example, when we define

```swift
let numbers: ObservableCollection([2, 3, 1])

let doubleNumbers = numbers.map { $0 * 2 }
let evenNumbers = numbers.filter { $0 % 2 == 0 }
let sortedNumbers = numbers.sort(<)
```

Modifying `numbers` will automatically update derived arrays:

```swift
numbers.append(4)

Assert(doubleNumbers.collection == [4, 6, 2, 8])
Assert(evenNumbers.collection == [2, 4])
Assert(sortedNumbers.collection == [1, 2, 3, 4])
```

With ReactiveUIKit, ObservableCollection containing an array can be bound to UITableView or UICollectionView. Just provide a closure that creates cells to the bindTo method.

```swift
let posts: ObservableCollection<[Post]> = ...

posts.bindTo(tableView) { indexPath, posts, tableView in
  let cell = tableView.dequeueCellWithIdentifier("PostCell", forIndexPath: indexPath) as! PostCell
  cell.post = posts[indexPath.row]
  return cell
}
```

Subsequent changes done to the ObservableCollection will then be automatically reflected in the table view.

## Operation

`Operation` type is used to represents asynchronous work that can fail. Operation constructor has one argument - an observer through which you send results.

```swift
func fetchImage(url: NSURL) -> Operation<UIImage, NSError> {
  return Operation { sink in
    Alamofire.request(.GET, url, parameters: [:]).response { request, response, data, error in
      if let error = error {
        sink.failure(error)
      } else {
        sink.next(UIImage(imageWithData: data!))
        sink.success()
      }
    }
  }
}
```

> You can also use function `create` to provide operations.

To create an operation that immediately succeeds do:

```swift
Operation<UIImage, NSError>.succeeded(with: UIImage(imageNamed: "dummyPhoto"))
```

Or the immediately fail:

```swift
Operation<UIImage, NSError>.failed(with: NSError(...))
```

Creating an operation does not do any work by itself. You have to start the operation to do the actual work. Operation is started when you register an observer with `observe` or `observeNext` methods:

```swift
fetchImage(url: ...).observeNext(on: Queue.Main.contex) { image in
  imageView.image = image
}
```

With ReactiveUIKit, you can also bind results of the operation to UIKit objects:

```swift
fetchImage(url: ...).bindNextTo(imageView)
```

> Method `bindNext` uses `observe` internally so binding will start the operation.

Each call to `observe` or `observeNext` method starts the operation all over again. To share results of a single operation, use `shareNext` method.

```swift
let image = fetchImage(url: ...).shareNext(on: Queue.Main.context)

image.bindTo(imageView1)
image.bindTo(imageView2)
```

> Method `shareNext` buffers results of the operation using `ActiveStream` type. Observing ActiveStream does not start any operation (no side-effects), rather it just replays what's in its buffer. 

Methods `observeNext`, `shareNext` or `bindNext` fire when the Operation sends next result, thus the suffix 'Next'. Operations can, however, fail without producing any result. To recover from a failure, you can use `flatMapError` transformation. For example, if image fetching fails, we would like to print the error and fallback to a default image:

```swift

fetchImage(url: ...)
  .flatMapError { error in
    print(error)
    return Operation.succeeded(with: UIImage(imageNamed: "dummy")!)
  }
  .bindNextTo(imageView)
```

## Streams

ReactiveKit is based on streams that conform to `StreamType` protocol. Basic requirement of a stream is that it produces events that can be observed.

```swift
public protocol StreamType {
  typealias Event
  func observe(on context: ExecutionContext, sink: Event -> ()) -> DisposableType
}
```

 When we observe streams, we have to think about threads. For example, when events that are being generated on a background thread as a result of a network response are used to update UI, we need those events on main queue. ReactiveKit uses simple concept of execution contexts inspired by [BrightFutures](https://github.com/Thomvis/BrightFutures).
> 
> When you want to receive events on the same thread on which they were generated, just pass `ImmediateExecutionContext`. When you want to receive them on a specific dispatch queue, just use `context` extension of dispatch queue wrapper type `Queue`, for example: `Queue.Main.context`. 

Observable, ObservableCollection and Operation are all streams. They differ in events they generate and whether their observation can cause side-effects or not.

`Observable` generates events of the same type it encapsulates. 

```swift
Observable<Int>(0).observe { (event: Int) in ... }
```

On the other hand, `ObservableCollection` generates events of `ObservableCollectionEvent` type. It's a struct that contains both the collection itself plus the change-set that describes performed operation.

```swift
ObservableCollection<[Int]>([0, 1, 2]).observe { (event: ObservableCollectionEvent<[Int]>) in ... }
```

```swift
public struct ObservableCollectionEvent<Collection: CollectionType> {
  
  public let collection: Collection

  public let inserts: [Collection.Index]
  public let deletes: [Collection.Index]
  public let updates: [Collection.Index]
}
```


Both Observable and ObservableCollection represent so called *hot streams*. It means that observing them does not perform any work and no side effects are generated. They are both subclasses of `ActiveStream` type. The type represents a hot stream that can buffer events. In case of Observable and ObservableCollection it buffers only one (latest) event, so each time you register an observer, it will be immediately called with the latest event - which is actually the current value of the observable.

`Operation` is a bit different. It's built upon generic `Stream` type. It represents *cold stream*. Cold streams don't do any work until they are observed. Once you register an observer, the stream executes underlying operation and side effect might be performed.

Operation generates events of `OperationEvent` type.

```swift
Operation<Int, NSError>(...).observe { (event: OperationEvent <Int, NSError>) in ... }
```

It's an enum defined like this:

```swift
public enum OperationEvent<Value, Error: ErrorType>: OperationEventType {
  case Next(Value)
  case Failure(Error)
  case Succes
```

Operation can send any number of `.Next` events followed by one terminating event - either a `.Success` or a `.Failure`. No events will ever be sent after terminating event has been sent.

You often don't observe operations with `observe` method, rather with `observeNext`.

### Transforming Streams

Great thing about streams is that we can transform them into another streams, for example:

```swift
func fetchAndBlurImage(url: NSURL) -> Operation<UIImage, NSError> {
  return fetchImage(url: url).map { $0.blurred() }
}
```

Or we can combine them, for example:

```swift
func authenticate() -> Operation<Token, NSError>
func fetchCurrentUser(token: Token) -> Operation<User, NSError>

...

authenticate().flatMap(.Latest) { token in
  return fetchCurrentUser(token)
}.observeNext(on: Queue.Main.context) { user in
  print("Authenticated as \(user)")
}
```

Just start typing `.` on any stream instance to see what's available.

## Requirements

* iOS 8.0+ / OS X 10.9+ / tvOS 9.0+ / watchOS 2.0+
* Xcode 7.1+

## Communication

* If you need help, use Stack Overflow. (Tag '**ReactiveKit**')
* If you'd like to ask a general question, use Stack Overflow.
* If you found a bug, open an issue.
* If you have a feature request, open an issue.
* If you want to contribute, submit a pull request.

## Why another FRP framework?

With Swift Bond I tried to simplify Model-View-ViewModel architecture in iOS apps, but as the framework grow it was becoming more and more reactive. That conflicted with its premise of being simple binding library.

ReactiveKit is a continuation of that project, but with different approach. It's based on streams inspired by ReactiveCocoa and RxSwift. It then builds upon those streams reactive types optimized for specific domains - `Operation` for asynchronous operations, `Observable` for observable variables and `ObservableCollection` for observable collections - making them simple and intuitive.

Main advantages over some other frameworks are clear separation of types that can cause side effects vs. those that cannot, less confusion around hot and cold streams (signals/producers), simplified threading and provided observable collection types with out-of-the box bindings for respective UI components.

## Installation

### CocoaPods

```
pod 'ReactiveKit', '~> 1.0'
pod 'ReactiveUIKit', '~> 1.0'
pod 'ReactiveFoundation', '~> 1.0'
```

### Carthage

```
github "ReactiveKit/ReactiveKit" 
github "ReactiveKit/ReactiveUIKit"
github "ReactiveKit/ReactiveFoundation"
```

## License

The MIT License (MIT)

Copyright (c) 2015 Srdan Rasic (@srdanrasic)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
