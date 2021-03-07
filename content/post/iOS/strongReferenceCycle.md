---
title: "Strong Reference Cycle in Swift"
date: 2021-03-07T22:16:03+08:00
draft: false
lastmod: 2021-03-07T22:16:03+08:00
tags: ["Swift", "ARC", "iOS"]
categories: ["iOS"]
---

Strong reference cycle experiments, on iOS 14.4 (XCode 12.4).

A good official article on [Automatic Reference Counting (ARC)](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html) in Swift.

<!--more-->

# A holding a reference to B, and B holding a reference to A

```swift
import UIKit

class ViewController: UIViewController {
    var b = B()
    
    override func viewDidLoad() {
        super.viewDidLoad()
                
        let a = A(b: self.b)
        self.b.a = a
    }
}

class A {
    let b: B
    
    init(b: B) {
        self.b = b
    }
}

class B {
    var a: A?
    
    init() {
        self.a = nil
    }
}
```

From the memory graph, it's pretty clear that object A and B have strong reference cycle.

![A to B (memory graph)](/blog/img/iOS/strongReferenceCycle/AToB.png)
![B to A (memory graph](/blog/img/iOS/strongReferenceCycle/BToA.png)

# The lazy var closure

## Will have strong reference cycle

```swift
import UIKit

class ViewController: UIViewController {
        
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let heading = HTMLElement(name: "h1")
        print(heading.asHTML())
    }
}

class HTMLElement {

    let name: String
    let text: String?

    lazy var asHTML: () -> String = {
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }

    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
        
        print("\(name) is being initialized")
    }

    deinit {
        print("\(name) is being deinitialized")
    }

}
```

![HTMLElement (memory graph](/blog/img/iOS/strongReferenceCycle/HTMLElement.png)

## Will NOT have strong reference cycle

But if you change `lazy var asHTML: () -> String` to `lazy var asHTML: String` and the code accordingly, the strong reference cycle won't happen!

```swift
import UIKit

class ViewController: UIViewController {
        
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let heading = HTMLElement(name: "h1")
        print(heading.asHTML)
    }
}

class HTMLElement {

    let name: String
    let text: String?

    lazy var asHTML: String = {
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }()

    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
        
        print("\(name) is being initialized")
    }

    deinit {
        print("\(name) is being deinitialized")
    }

}

```
