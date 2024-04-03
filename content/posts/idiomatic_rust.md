+++
title = 'idiomatic rust way'
date = 2024-04-03T15:28:46+08:00
draft = false
+++

## 遍历可遍历对象(Iterator)
The core of [`Iterator`](https://doc.rust-lang.org/std/iter/trait.Iterator.html "trait std::iter::Iterator")：
```Rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```
`for` 循环是迭代器的语法糖
#### 普通方式：
```Rust
for i in 1..10 {
	println!("{}", i);
}
```

#### 遍历数组：
```Rust
// 数组实现了 IntoIterator trait，隐式调用into_iter() method
let arr = [1, 2, 3];
for i in arr {
    println!("{i}");
}

// enumerate
let v = vec![100, 32, 57];
for (i, v) in v.iter().enumerate() {
    println!("{} is {}", i, v);
}
```

#### 链式方法调用
- `into_iter` 会夺走所有权
- `iter` 是借用
- `iter_mut` 是可变借用

```Rust
let v1: Vec<i32> = vec![1, 2, 3];
// map 是迭代器适配器，collect和sum是消费性适配器
let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();
```
## 对 HashMap 的操作
* 更新：*m.entry(key).or_insert(0) += 1;
* 读取：*m.get(&key).unwrap()
