## Swift. Speed up compilation.

I’ve had an honor to join an old mature project fully written in Swift. The project has a great team and high quality code base to pleasure to work with.

The code base is huge. We all (iOS devs) love to work with value types because of its nature of immutability and this project is not exception - value types are everywhere. But on iOS we cannot escape from using reference data types like classes as we use Apple system frameworks with public APIs of which are written in objective-c and have to carry all burden of inheritance from it. 

As you may know the swift compiler analyses our code to produce very efficient machine code. Swift syntax allows to us to write very short precise clear code, but that goes with drawbacks as slow compilation time for example. It is slow as swift has a lot of syntax sugar, automatic type inference feature and other magic stuff. All this gonna be handled by compiler during compilation what causes slow compilation.

Once I heard that if you mark your class as final that will reduce compile time of this class and sometimes significantly. Oh yeah! But why? Let's talk about message dispatch mechanism or how the class's methods are actually called. 

As we all know in objective-c all method calls done through runtime message dispatch - messages (selectors) are being send to objects at runtime. If object can respond to this message then method will be called else unrecognized selector runtime error will be thrown. Objective-C compiler knows almost nothing about code, it doesn't even know is object has the being calling method or not. Because of this objective-c compiler do not produce highly efficient machine code as it doesn't know what is going on there. All this information begins available only in runtime and can be optimized at this time as actually objective-c does, for example cashes method poiters of vtable and etc. That being said objective-c code not efficient as Swift one because of its nature and we can do nothing about this.

Now let's talk about Swift's methods calls. Will talk about only pure Swift methods invocations, not where swift tightly coupled with objective-c as example via inheritance from NSObject because there begins working objective-c's message dispatch. Pure swift has two ways to call a method that is dynamic dispatch or static dispatch. Dynamic dispatch starts working when compiler does not exactly know what type of object for called method gone be used and the runtime will go through all inheritance tree to find needed method to call.

```swift
// Instead of base class we can declare a protocol Animal, the effect will be the same.

class Animal {
    func sayName() {
        print("I'm just Animal")
    }
}

class Cat: Animal {
    override func sayName() {
        print("I'm a Cat!")
    }
}

func examineAnimal(_ animal: Animal) {

    // Compiler might not know is there Animal object or its subclass, 
    // to find out the runtime will go through all inheritance tree to find needed method.

    animal.sayName()
}

examineAnimal(Animal()) // prints "I'm just Animal"
examineAnimal(Cat()) // prints "I'm a Cat!"
```

From code above you can see what means dynamic dispatch and it's a bit slow agains static dispatch. If we change the code to this


```swift
class Animal {
    func sayName() {
        print("I'm just Animal")
    }
}

func examineAnimal(_ animal: Animal) {
    animal.sayName()
}

examineAnimal(Animal())
```

the compiler can determine that there are no any subclasses for Animal and can enable here static dispatch, but to do this the compiler has to analyze all your code base to be sure that there are no subclasses and this process can slow the compiler down.

The static dispatch means that calling object is already known at compile time and there no needs to traverse all inheritance tree to find it. It is very fast method of calling because object and method memory addresses already set by compiler and the runtime only needs to use it without any additional work.

When we are sure that there are no subclasses for some class we can help the compiler to know about it much earlier and do not trigger full code base analysis and save us some time. To do this just use Swift's keyword final.

```swift
final class Animal {
    func sayName() {
        print("I'm just Animal")
    }
}

func examineAnimal(_ animal: Animal) {
    animal.sayName()
}

examineAnimal(Animal())
```

Found this magic word the compiler sure there no subclasses and will directly use object's and method's addresses to set them to the binary.


Okay, back to our project. The project has thousands of swift files, most of them are swift's structs and protocols. I've found about 650 classes, some of them have subclasses, but most of them (~600) do not and no one was marked as final. It took about 20 minutes to mark 600 classes as final (find/replace by rexep). Then I run a set of project builds to measure its time. Here are my results with final classes and without it.

#### Environment settings:

- MacOS 10.14.3 (18D109)
- MacBook Pro (15-inch, 2016) / 2.6 GHz Intel Core i7 / 16 GB 2133 MHz LPDDR3
- Xcode 10.1 / Build version 10B61
- Apple Swift version 4.2.1 (swiftlang-1000.11.42 clang-1000.11.45.1)
- Xcode new build system
- All dependencies are excluded from build
- XCAssets and Storyboards/Xibs are excluded from build
- Build for only one active architecture (x86_64)
- DEBUG build (as we use it mostly during development)

Before every /Build I was deleting the directory _/Library/Developer/Xcode/DerivedData/my_project-hfwwsqmzzsigtuazoqlonulxiafe/Build/Intermediates.noindex/my_project.build/Debug-iphonesimulator/my_project.build/Objects-normal/x86_64_ to trigger only swift source recompilation of Xcode active target. The results are below.

|final class|class|
|:-------------:|:-------------:|
| 109 seconds   | 119 seconds   |
| 110 seconds   | 119 seconds   |
| 110 seconds   | 119 seconds   |
| 106 seconds   | 121 seconds   |
| 109 seconds   | 122 seconds   |

As we can see marking all classes as final we got 8%~10% compilation time speed up. Cheers!









