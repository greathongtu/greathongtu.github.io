+++
title = 'Visitor'
date = 2024-04-27T19:03:44+08:00
draft = false
+++

- Vistor 模式通过双重分发(double dispatch) 来实现在不更改（不添加新的编译时操作）Element类层次结构的前提下，在运行时透明地为类层次结构上的各个类动态添加新的操作。
- Double Dispatch: 第一个为accept方法的多态辨析（哪个element调用的accept）；第二个为visitElementX方法的多态辨析（哪个visitor）。
- Vistor 模式的最大缺点在于拓展类层次结构（增加新的Element 子类）会导致 Visitor 类的改变。因此 Visitor 模式适用于“Element 类层次结构稳定，而其中的操作却经常面临频繁改动”。

C++ 代码示例：
```cpp
#include <iostream>
#include <array>

using namespace std;
class class_a;
class class_b;

class Visitor {
public:
  virtual void visit_a(class_a* element_a) = 0;
  virtual void visit_b(class_b* element_b) = 0;
};

class Component {
public:
  virtual void accept(Visitor* visitor) = 0;
};

class class_a : public Component {
  void accept(Visitor* visitor) override {
    visitor->visit_a(this);
  }
};

class class_b : public Component {
  void accept(Visitor* visitor) override {
    visitor->visit_b(this);
  }
};

// 在此之上是稳定不变的部分
// ====================================
// 在此之下是变化的部分

class MyVisitor1 : public Visitor {
  void visit_a(class_a* ea) {
    cout << "MyVisitor1 visit_a" << endl;
  }
  void visit_b(class_b* eb) {
    cout << "MyVisitor1 visit_b" << endl;
  }
};

class MyVisitor2 : public Visitor {
  void visit_a(class_a* ea) {
    cout << "MyVisitor2 visit_a" << endl;
  }
  void visit_b(class_b* eb) {
    cout << "MyVisitor2 visit_b" << endl;
  }
};

int main() {
  std::array<Component *, 2> components = {new class_a, new class_b};
  Visitor* v1 = new MyVisitor1;
  Visitor* v2 = new MyVisitor2;
  for (auto* pc : components) {
    pc->accept(v1);
    pc->accept(v2);
  }
}
```
Rust 代码示例：
```Rust
trait Item {
    fn accept(&self, visitor: &dyn Visitor);
}

struct Item1 {}
struct Item2 {}

impl Item for Item1 {
    fn accept(&self, visitor: &dyn Visitor) {
        visitor.visit_item1(self);
    }
}

impl Item for Item2 {
    fn accept(&self, visitor: &dyn Visitor) {
        visitor.visit_item2(self);
    }
}

trait Visitor {
    fn visit_item1(&self, item1: &Item1);
    fn visit_item2(&self, item2: &Item2);
}

// 在此之上是稳定不变的部分
// ====================================
// 在此之下是变化的部分

struct Item1Visitor;

impl Visitor for Item1Visitor {
    fn visit_item1(&self, _item1: &Item1) {
        println!("Item1Vistor visit_item1");
    }
    fn visit_item2(&self, _item2: &Item2) {
        println!("Item1Vistor visit_item2");
    }
}

fn main() {
    let mut elements: Vec<&dyn Item> = Vec::new();
    let item1 = Item1 {};
    let item2 = Item2 {};
    elements.push(&item1);
    elements.push(&item2);

    let visitor = Item1Visitor;
    for element in elements {
        element.accept(&visitor);
    }
}

```
