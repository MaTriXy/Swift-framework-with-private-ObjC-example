# Creating Swift framework with a private Objective-C members. The "Good", the "Bad", and the "Ugly".

I was recently working on a closed source Swift framework. Unluckily, some parts of the code were in Objective-C (relying on pure C libraries). In this article I will highlight some of the problems when having frameworks with mixed Swift/ObjC code. I will show some approaches I tried when struggling to make ObjC code internal. Making it invisible to the framework users, but still accessible from Swift code gave me a serious headache. And I will share a solution that finally worked out in my case.

# The Problem

There are a lot of articles and tutorials about Swift/ObjC interoperability, but they rarely focus on framework targets. It seems that even when you have everything in place, it is still close to impossible to effectively hide your Objective-C part from framework users (at least as long as you want to expose it to the Swift part). Or is it?

Let's assume you created a framework, named **MyFramework**, with following members:

```objc
// MyPrivateClass.h

@interface MyPrivateClass: NSObject

- (void) doSomethingInternalWithSecretAttribute:(NSInteger)attribute;

@end
```

```swift
// MyPublicClass.swift

public class MyPublicClass {
    private let privateClass: MyPrivateClass // We need private ObjC member

    ...

    public func doSomething() {
        privateClass.doSomethingInternal(withSecretAttribute: 314)
    }
}
```

Our Swift class depends on Objective-C class, so it needs to know about it somehow. But we don't want the client app to be able to see our ObjC internals:

```swift
// Somewhere in the wild west
import MyFramework

...

// This should work
let publicClass = MyPublicClass()
publicClass.doSomethingWithPrivateClass()

...

// This should not be possible, nor even compile,
// as it should not be able to see MyPrivateClass !!!
let privateClass = MyPrivateClass()
privateClass.doSomethingInternal(withSecretAttribute: 13)
```

Seems easy, right? Well, it is actually harder than it looks.

> ### TL;DR
> - we have a mixed Objective-C, C (or C++), Swift framework target
> - we need to expose an Objective-C members to Swift code internally
> - we don't want framework users, to see our internal ObjC members

# Solutions

## The Good - yet not working

By that I mean "default" Swift ObjC interoperability. Just for a short recap:

- In order to expose **Swift to Objective-C**:
  - target needs to define module
  - Swift member should be marked as `@objc`
  - Swift member should inherit from NSObject
- In order to expose **Objective-C to Swift**:
  - place ObjC imports in bridging header file (if available)
  - place ObjC imports in umbrella header (for frameworks)

You can read more about it here:

- [Apple: Importing Swift into Objective-C](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/importing_swift_into_objective-c)
- [Apple: Importing Objective-C into Swift](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/importing_objective-c_into_swift)
- [Jen Sipila article about Swift interoperability](https://medium.com/ios-os-x-development/swift-and-objective-c-interoperability-2add8e6d6887)

There is one significant problem with this approach:

> "Swift sees every header you expose **publicly** in your umbrella header. The contents of the Objective-C files in that framework are automatically available from any Swift file within that framework target, with no import statements."

So it **does not provide desired behaviour**. Moreover, I noticed that quite often it leads to a chain of "including non-modular header" issues. That literally makes you rewrite half of the headers. It is still doable, as long as you don't import any header that is out of your control, for example headers from some static library. Summing up:

**Pros:**
+ Its intended way to do Swift ObjC interoperability

**Cons:**
- All Objective-C members exposed to Swift are also publicly visible
- Common issues with importing non-modular headers

## The Bad - a private module maps that almost work

When searching for a possible solution, this is one of the most popular. It relies on creating **private modulemap** file, that will define **explicit submodule**. I don't want to get too much into details how to do it, here are some useful links:

- [stackoverflow: How to import private framework headers in a Swift framework?](https://stackoverflow.com/questions/28746214/how-to-import-private-framework-headers-in-a-swift-framework)
- [https://code.i-harness.com/en/q/21e3c64](https://code.i-harness.com/en/q/21e3c64)
- [Realm Academy](https://academy.realm.io/posts/marius-rackwitz-challenges-building-swift-framework/)
- [Omar Abdelhafith blog](http://nsomar.com/project-and-private-headers-in-a-swift-and-objective-c-framework/)

In quick words - you can manually define an explicit submodule with a private header files, named for example **MyFramework.Private**. Then you access it from your Swift code as following:

```swift
// MyPublicClass.swift
import MyFramework.Private // Required to see "private" header

public class MyPublicClass {
    private let privateClass: MyPrivateClass
    ...
```

**It is not really private though** as it would work also for the framework user:

```swift
// Somewhere in the wild west
import MyFramework
import MyFramework.Private

...

// This should not be possible, nor even compile,
// as it should not be able to see MyPrivateClass
let privateClass = MyPrivateClass()
privateClass.doSomethingInternal(withSecretAttribute: 13)
```

**Framework users still can access "private" members**. It is a bit harder, since importing main module does not reveal "private" members immediately. And **it requires additional import statement to access "private" members**. As a framework creator you can emphasise that it is not safe nor intended to use it. But you cannot hide it.

To sum it up:

**Pros:**
+ It gives some kind of control over what is visible to the user by default
+ Allows to emphasise, that "private" submodule is for internal use only and it should not be used directly

**Cons:**
- All Objective-C members exposed to Swift **are still public**
- It requires manual module maps, which are harder to setup, and easier to misuse

## The Ugly - a solution that actually works

In my case, the modulemaps were not enough. The research output wasn't promising. It seemed that **there is no way to expose Objective-C headers privately**, inside the same framework target only, without making them more or "less" public. **And it is true**.

You might wonder what I did have in mind then, when writing about finding **solution that works**. Let me share a part of a discussion I had with myself:

> **Me:** *"Ok, let's think again, what do you actually want to achieve?"*
>
> **Me2:** *"I want to expose Objective-C headers to Swift, but only inside the framework target, without exposing them to the framework users."*
>
> **Me:** *"Do you? Is that __exactly__ what you need?"*
>
> **Me2:** *"Well, I have ObjC classes that are meant to be internal, but I want to use them from Swift code."*
>
> **Me:** *"OK. How would you actually 'use' this classes?"*
>
> **Me2:** *"Call some methods, access variables?"*
>
> **Me:** *"Do you need to know the 'class' to do that?"*
>
> **Me2:** *"Well, I only need to know that there is a method or a variable, so protocol should be fine."*
>
> **Me:** *"And if you invert the problem? Can you have an internal Swift member and use it from Objective-C"*
>
> **Me2:** *"It seems that the answer is yes, with some restrictions of course"*
>
> **Me:** *"OK. Can you make then the private Objective-C member adopt an internal Swift protocol?"*
>
> **Me2:** *"Seems so... __Does it mean I don't need to know about any Objective-C class, as I interact only with things adopting Swift protocols?__*
>
> **Me:** *"Well, let's try."*

### Step 1: Create Swift protocol matching ObjC member

Let's update the framework a bit according to that idea. Swift code does not have to see the Objective-C part at all. But it can operate on anything adopting protocol matching our ObjC class features:

```swift
// ObjectiveCToSwift.swift

@objc(MyPrivateClassProtocol) // Under this name this will cross Swift->ObjC boundary
internal protocol MyPrivateClass {
    init()
    func doSomethingInternal(withSecretAttribute: Int)
}
```

### Step 2: Expose internal Swift protocol

Because `MyPrivateClassProtocol` is internal, it would not be a part of `MyFramework-Swift.h` by default. So let's created additional header for making a link between all our internal Swift and ObjC members:

```objc
// SwiftToObjectiveC.h

#ifndef SwiftToObjectiveC_h
#define SwiftToObjectiveC_h

SWIFT_PROTOCOL_NAMED("MyPrivateClassProtocol")
@protocol MyPrivateClassProtocol

- (nonnull instancetype)init;
- (void)doSomethingInternalWithSecretAttribute:(NSInteger)attribute;

@end

#endif /* SwiftToObjectiveC_h */
```

> **Note:** If you are unsure how should you fill this header, you can make Swift members public and build. That will generate ModuleName-Swift.h header in derived data. Inspect it and it will give you some sense of how to do it. Then make it internal again.

### Step 3: Adopt Swift protocol in ObjC

Now as we are able to see Swift protocol in Objective-C as `MyPrivateClassProtocol`, we can adopt it.

```objc
// MyPrivateClass.h
#import <Foundation/Foundation.h>
#import <SwiftToObjectiveC.h>

@interface MyPrivateClass : NSObject<MyPrivateClassProtocol>

- (void) doSomethingInternalWithSecretAttribute:(NSInteger)attribute;

@end
```

That leaves us with one last final problem.

### Step 4: How to create instances

> **Me2:** *"OK, I theoretically can use it. But again, how do I get an instance of it if I don't know about a class adopting protocol?"*
>
> **Me:** *"Remember that factory pattern? Or a dependency injection containers you once wrote?"*

The problem with the solution above is that we simply cannot instantiate a new instance of a Protocol without using the init on a concrete class. Or can we? Let's consider this:

```swift
protocol SomeProtocol {
    init()
}
class SomeClass: SomeProtocol {
    ...
}

// This will work
let instance: SomeProtocol = SomeClass.init()

// This won't work
let instance: SomeProtocol = SomeProtocol.init()

// Surprisingly, this also works
var type: SomeProtocol.Type!                // Expose from Swift
type = SomeClass.self                       // Move it to ObjC!
let instance: SomeProtocol = type.init()    // Use in Swift
```

Let's focus on last part. We can create a helper factory class, that holds MyPrivateClass.Type:

```swift
@objc(SwiftFactory)
internal class Factory {
    private static var privateClassType: MyPrivateClass.self! // Protocol

    // Expose registering class for protocol to ObjC
    @objc static func registerMyPrivateClass(type: MyPrivateClass.self) {
        privateClassType = type
    }

    // Factory methods
    func createMyPrivateClass() -> MyPrivateClass {
        return privateClassType.init()
    }
}
```

> **Note:** In Swift part, MyPrivateClass is a protocol, while in ObjC it is a class.

Factory above would be exposed to Objective-C in the same way as the MyPrivateClass Protocol definition (so internally).

The one last thing that needs to be done is to register Objective-C class to be used by factory. We need to do it before it could possibly be used.

Luckily, Obj-C runtime has something just up to the job:

```objc
// MyPrivateClass.m
#import "MyPrivateClass.h"

@implementation MyPrivateClass

@synthesize privateProperty;

+ (void)load {
    // This is called once, when module is being loaded,
    // "Invoked whenever a class or category is added to the Objective-C
    // runtime; implement this method to perform class-specific behavior
    // upon loading."
    [Factory registerPrivateClassTypeWithType:[MyPrivateClass class]];
}

- (void)doSomethingInternalWithSecretAttribute:(NSInteger)attribute {
    NSLog(@"INTERNAL METHOD CALLED WITH SUCCESS %ld", (long)attribute);
}

@end
```

So it seems, that **we are able to use Objective-C members in Swift without exposing ObjC code to Swift at all!** All we need to do is to expose Swift part to Objective-C, which we can do internally 😎.

Let's sum it up:

**Pros:**
+ **It effectively hides Objective-C members** from framework users
+ you are forced to interface stuff. This is usually a good thing, increasing overall testability

**Cons:**
+ You need some additional members to cross Swift-ObjC boundary
+ It's a manual job to generate Swift protocol for every ObjC member

## Summary

The example project is available on the github: [https://github.com/amichnia/Swift-framework-with-private-ObjC-example](https://github.com/amichnia/Swift-framework-with-private-ObjC-example)

The whole approach requires some manual work and forces to slightly redesign how you initialize your dependencies. You need to generate protocols, manually bridge it to objc, and also maintain factory along the way. But this is the only way so far to have Objective-C members truly internally exposed to the Swift code.

> **Note:** In the example I used factory that allows you to register a type, and then I used init on that type. You can play with this approach a bit. If you don't like to add inits to protocols, you can try to use closure/block as a factory: `static var createPrivateClass: (() -> MyPrivateClass)!`

### Thank you for reading

If you liked it, feel free to 👏
