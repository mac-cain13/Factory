![](https://github.com/hmlongco/Factory/blob/main/Logo.png?raw=true)

A new approach to Container-Based Dependency Injection for Swift and SwiftUI.

## Why Something New?

The first dependency injection system I ever wrote was [Resolver](https://github.com/hmlongco/Resolver). This open source project, while quite powerful and still in use in many applications, suffers from a few drawbacks.

1. Resolver requires pre-registration of all services up front. 
2. Resolver uses type inference to dynamically find and return registered services from a container.

The first drawback is relatively minor. While preregistration could lead to a performance hit on application launch, in practice the process is usually quick and not normally noticable.

No, it’s the second one that’s somewhat more problematic.

Failure to find a matching type can lead to an application crash if we attempt to resolve a given type and a matching registration is not found. In real life that isn’t really a problem as such a thing tends to be noticed and fixed rather quickly the very first time you run a unit test or the second you run the application to see if your newest feature works.
 
 But... could we do better? That question lead me on a quest for compile-time type safety. Several other systems have attempted to solve this, but I didn't want to have to add a source code scanning and generation step to my build process, nor did I want to give up a lot of the control and flexibility inherent in a run-time-based system.
 
 I also wanted something simple, fast, clean, and easy to use.
 
 Could I have my cake and eat it too?
 
 ## Features
 
 Factory is strongly influenced by SwiftUI, and in my opinion is highly suited for use in that environment. Factory is...
 
 * **Safe:** Factory is compile-time safe; a factory for a given type *must* exist or the code simply will not compile.
 * **Flexible:** It's easy to override dependencies at runtime and for use in SwiftUI Previews.
 * **Powerful:** Like Resolver, Factory supports application, cached, shared, and custom scopes, custom containers, arguments, decorators, and more.
 * **Lightweight:** With all of that Factory is slim and trim, coming in under 300 lines of code.
 * **Performant:** Little to no setup time is needed for the vast majority of your services, resolutions are extremely fast, and no compile-time scripts or build phases are needed.
 * **Concise:** Defining a registration usually takes just a single line of code. Same for resolution.
 * **Tested:** Unit tests ensure correct operation of registrations, resolutions, and scopes.
 * **Free:** Factory is free and open source under the MIT License.
 
 Sound too good to be true? Let's take a look.
  
 ## A Simple Example
 
 Most container-based dependency injection systems require you to define in some way that a given service type is available for injection and many reqire some sort of factory or mechanism that will provide a new instance of the service when needed.
 
 Factory is no exception. Here's a simple dependency registraion.
 
```swift
extension Container {
    static let myService = Factory { MyService() as MyServiceType }
}
```
Unlike Resolver which often requires defining a plethora of nested registration functions, or SwiftUI, where defining a new environment variable requires creating a new EnvironmentKey and adding additional getters and setters, here we simply add a new `Factory` to the default container. When called, the factory closure is evaluated and returns an instance of our dependency. That's it.

Injecting and using the service where needed is equally straightforward. Here's one way to do it.

```swift
class ContentViewModel: ObservableObject {
    @Injected(Container.myService) private var myService
    ...
}
```
Here our view model uses one of Factory's `@Injected` property wrappers to request the desired dependency. Similar to `@EnvironmentObject` in SwiftUI, we provide the property wrapper initializer with a reference to a factory of the desired type and it handles the rest.

And that's the core mechanism. In order to use the property wrapper you *must* define a factory. That factory *must* return the desired type when asked. Fail to do either one and the code will simply not compile. As such, Factory is compile-time safe.

 ## Factory

Similar to a `View` in SwiftUI, a `Factory` is a lightweight struct that exists to define and manage a specific dependency. Just provide it with a closure that constructs and returns an instance of your dependency or service, and Factory will handle the rest.

```swift
static let myService = Factory { MyService() as MyServiceType }
```

The type of a factory is inferred from the return type of the closure. Here's we're casting `MyService` to the protocol it implements, so any dependency returned by this factory will always conform to `MyServiceType`. 

We can also get the same result by explicitly specializing the generic Factory as shown below. Both the specialization and the cast are equivalent and provide the same result.

```swift
static let myService = Factory<MyServiceType> { MyService() }
```

Do neither one and the factory type will always be the returned type. In this case it's `MyService`.

```swift
static let myService = Factory { MyService() }
```

Due to the lazy nature of static variables, no factory is instantiated until it's referenced for the first time. Contrast this with Resolver, which forced us to run code to register *everything* prior to resolving anything.

Finally, note that it's possible to bypass the property wrapper and talk to the factory yourself in a *Service Locator* pattern.

```swift
class ContentViewModel: ObservableObject {
    // dependencies
    private let myService = Container.myService()
    private let eventLogger = Container.eventLogger()
    ...
}
```
Just call the desired specific factory as a function and you'll get an instance of its managed dpendency. It's that simple.

*You can access the factory directly or the property wrapper if you prefer, but either way for clarity I'd suggest grouping all of a given object's dependencies in a single place near the top of the class and marking them as private.*

## Mocking and Testing

If we go back and look at our view model code one might wonder why we've gone to all of this trouble? Why not simply say `let myService = MyService()` and be done with it? 

Or keep the container idea, but write something similar to this…

```swift
extension Container {
    static var myService: MyServiceType { MyService() }
}
```

Well, the primary benefit one gains from using a container-based dependency injection system is that we're able to change the behavior of the system as needed. Consider the following code:

```swift
struct ContentView: View {
    @StateObject var model = ContentViewModel()
    var body: some View {
        Text(model.text())
            .padding()
    }
}
```

Our ContentView uses our view model, which is assigned to a StateObject. Great. But now we want to preview our code. How do we change the behavior of `ContentViewModel` so that its `MyService` dependency isn't making live API calls during development? 

It's easy. Just replace `MyService` with a mock that also conforms to `MyServiceType`.

```swift
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        let _ = Container.myService.register { MockService2() }
        ContentView()
    }
}
```

Note the line in our preview code where we’re gone back to our container and registered a new closure on our factory. This function overrides the default factory closure.

Now when our preview is displayed `ContentView` creates a `ContentViewModel` which in turn has a dependency on `myService` using the `Injected` property wrapper. 

And when the wrapper asks the factory for an instance of `MyServiceType` it now gets a `MockService2` instead of the `MyService` type originally defined.


This is a powerful concept that lets us reach deep into a chain of dependencies and alter the behavior of a system as needed.

But we're not done yet. 

Factory has quite a few more tricks up its sleeve...

## Scope

If you've used Resolver or some other dependency injection system before then you've probably experienced the benefits and power of scopes.

And if not, the concept is easy to understand: Just how long should an instance of an object live?

You've no doubt stuffed an instance of a class into a variable and created a singleton at some point in your career. This is an example of a scope. A single instance is created and then used and shared by all of the methods and functions in the app.

This can be done in Factory just by adding a scope attribute.

```swift
extension Container {
    static let myService = Factory(scope: .singleton) { MyService() as MyServiceType }
}
```
Now whenever someone requests an instance of `myService` they'll get the same instance of the object as everyone else.

Unless altered, the default scope is `unique`; every time the factory is asked for an instance of an object it will get a new instance of that object.

Other common scopes are `cached` and `shared`. Cached items are persisted until the cache is reset, while shared items exist just as long as someone holds a strong reference to them. When the last reference goes away, the weakly held shared reference also goes away.

You can also add your own special purpose caches to the mix. Try this.

```swift
extension Container.Scope {
    static var session = Cached()
}

extension Container {
    static let authenticatedUser = Factory(scope: .session) { AuthenticatedUser() }
    static let profileImageCache = Factory(scope: .session) { ProfileImageCache() }
}
```
Once created, a single instance of `AuthenticatedUser` and `ProfileImageCache` will be provided to anyone that needs one... up until the point where the session scope is reset, perhaps by a user logging out.

```swift
func logout() {
    Container.Scope.session.reset()
    ...
}
```
Scopes are powerful tools to have in your arsenal. Use them.

## Constructor Injection

At times we might prefer (or need) to use a technique known as *constructor injection* where dependencies are provided to an object upon initialization. 

That's easy to do in Factory. Here we have a service that needs an instance of `MyServiceType`.

```swift
extension Container {
    static let constructedService = Factory { ConstructedService(service: myService()) }
}
```
All of the factories in a container are visible to the other factories in that container. Just call the needed factory as a function and the dependency will be provided.

## Dynamic Registration

With Factory registrations can be performed at any time. Consider.

```swift
extension Container {
    static let userProviding = Factory<UserProviding?> { nil }
}

func authenticated(with user: User) {
    ...
    Container.userProviding.register { UserProvider(user: user) }
    ...
}

func logout() {
    ...
    Container.userProviding.reset()
    ...
}
```
Now any view model or service that needs an instance of an authenticated user will receive one (or nothing if no user is authenticated).

Note that Factory is *thread-safe.* Registrations and resolutions lock and unlock the containers and caches as needed.

Also note that calling register also *removes any cached dependency from its associated scope.* This ensures that any new dependency injection request performed from that point on will always get the most recently defined instance of an object.

## Custom Containers

In a large project you might want to segregate factories into additional, smaller containers.

```swift
class OrderContainer: SharedContainer {
    static let optionalService = Factory<SimpleService?> { nil }
    static let constructedService = Factory { MyConstructedService(service: myServiceType()) }
    static let additionalService = Factory(scope: .session) { SimpleService() }
}
```
Just define a new container derived from `SharedContainer` and add your factories there. You can have as many as you wish, and even derive other containers from your own. 

```swift
class PaymentsContainer: OrderContainer {
    static let paymentsServiceType = Factory<PaymentsServiceType> { PaymentsService(service: myServiceType()) }
}
```

While a container *tree* makes dependency resolutions easier, don't forget that if need be you can reach across containers simply by specifying the full container.factory path.

```swift
class PaymentsContainer: SharedContainer {
    static let anotherService = Factory { AnotherService(OrderContainer.optionalService()) }
}
```

## SharedContainer

Note that you can also add your own factories to the root `SharedContainer` class. Anything added there will be visible  and available to every container in the system.

```swift
extension SharedContainer {
    static let api = Factory<APIServiceType> { APIService() }
}
```
Note that any registrations defined with your app are managed by `SharedContainer`. More on this later.

## Setup

If we have several mocks that we use all of the time in our previews, we can also add a setup function to the container to make this easier.

```swift
extension Container {
    static func setupMocks() {
        myService.register { MockServiceN(4) }
        sharedService.register { MockService2() }
    }
}

struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        let _ = Container.setupMocks()
        ContentView()
    }
}
```

## Reset

You can also reset a registration to bring back the original factory closure. Or, if desired, you can reset everything back to square one with a single command.

```Swift
Container.myService.reset() // single
Container.Registrations.reset() // all 
```

The same applies to scope management. You can reset a single cache, or all of them if desired. This includes any caches you might have added, like the `session` scope we added above.

```Swift
Container.Scope.cached.reset() // single
Container.Scope.reset() // all scopes except singletons
Container.Scope.reset(includingSingletons: true) // all including singletons
```
The `includingSingletons` option must be explicitly specified in order to reset singletons. You have the power. Use it wisely.

## Xcode Unit Tests

Finally, Factory has a few more provisions added to make unit testing eaiser. In your unit test setUp function you can *push* the current state of the registration system and then register and test anything you want.

```swift
final class FactoryCoreTests: XCTestCase {

    override func setUp() {
        super.setUp()
        Container.Registrations.push()
        Container.setupMocks()
     }

    override func tearDown() {
        super.tearDown()
        Container.Registrations.pop()
    }
    
    func testSomething() throws {
        Container.myServiceType.register(factory: { MockService() })
        let model = Container.someViewModel()
        XCTAssertTrue(model.isLoaded)
        ...
    }
}
```

Then in your tearDown function simply *pop* your changes to restore everything back to the way it was prior to running that test suite.


## Resolver

Factory will probably mark the end of Resolver. I learned a lot from that project, and it even won me an [Open Source Peer Bonus from Google](https://opensource.googleblog.com/2021/09/announcing-latest-open-source-peer-bonus-winners.html). (I always thought it a bit strange for an iOS developer to get an award from Google, but there you have it.)

But Factory is smaller, faster, cleaner and all in all a much better solution than Resolver could ever be.

## Installation

Factory is available as a Swift Package. Just add it to your projects.

## License

Factory is available under the MIT license. See the LICENSE file for more info.

## Author

Factory was designed, implemented, documented, and maintained by [Michael Long](https://www.linkedin.com/in/hmlong/), a Senior Lead iOS engineer at [CRi Solutions](https://www.clientresourcesinc.com/solutions/). CRi is a leader in developing cutting edge iOS, Android, and mobile web applications and solutions for our corporate and financial clients.

* Email: [mlong@clientresourcesinc.com](mailto:mlong@clientresourcesinc.com)
* Twitter: @hmlco

He was also one of Google's [Open Source Peer Reward](https://opensource.googleblog.com/2021/09/announcing-latest-open-source-peer-bonus-winners.html) winners in 2021 for his work on Resolver.

## Additional Resources

* [Resolver: A Swift Dependency Injection System](https://github.com/hmlongco/Resolver)
* [Inversion of Control Design Pattern ~ Wikipedia](https://en.wikipedia.org/wiki/Inversion_of_control)
* [Inversion of Control Containers and the Dependency Injection pattern ~ Martin Fowler](https://martinfowler.com/articles/injection.html)
* [Nuts and Bolts of Dependency Injection in Swift](https://cocoacasts.com/nuts-and-bolts-of-dependency-injection-in-swift/)
* [Dependency Injection in Swift](https://cocoacasts.com/dependency-injection-in-swift)
* [Swift 5.1 Takes Dependency Injection to the Next Level](https://medium.com/better-programming/taking-swift-dependency-injection-to-the-next-level-b71114c6a9c6)
