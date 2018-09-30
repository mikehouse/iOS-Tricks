## The access speed to local vs. global/shared reference type variable

We all know that the access to a variable have some cost if it's not inlined or not somehow optimized by compiler. There is one mechanism to speed it up which is simple to use. Let's check.

Make some dummy class that we will call

```swift
class Transformer {
    func transform(_ value: Int) -> String {
        return String(value)
    }
}
```

Firstly do the test where we access some public mutable variable:


```swift
class ViewController: UIViewController {

    var transformar = Transformer()

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)

        DispatchQueue.global(qos: .userInitiated).async {

            let start = CFAbsoluteTimeGetCurrent()

            for n in 0..<50_000_000 {
                let _ = self.transformar.transform(n)
            }

            print(CFAbsoluteTimeGetCurrent() - start)
        }
    }
}
```

The average result is `13.683467030525208` seconds.

Second examle will use a local variable (same effect can be achieved making variable `let` or `private`):

```swift
class ViewController: UIViewController {

    var transformar = Transformer()

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)

        DispatchQueue.global(qos: .userInitiated).async {

            let start = CFAbsoluteTimeGetCurrent()
            let transformar = self.transformar

            for n in 0..<50_000_000 {
                let _ = transformar.transform(n)
            }

            print(CFAbsoluteTimeGetCurrent() - start)
        }
    }
}
```

The average result is `4.625336050987244` seconds.

#### Takeaways

If you do some heavy work in a loop where you access a lot of mutable shared global variables and it's safe to make them local referenced than use this tip to speed your code up.

Notes: make sure that it

- runs on real device
- compiled with `-O0` flag (not optimized)
