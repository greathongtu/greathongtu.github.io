+++
title = 'Zed Rope'
date = 2024-09-06T20:47:19+08:00
draft = true
+++

# Zed 编辑器中的 SumTree 和 Rope 数据结构

代码编辑器最重要的东西就是如何表示文本了。朴素的想法是直接使用 string 数据结构。但作为使用连续内存的数据结构，在面对中间位置的数据插入删除时代价过大。并且更复杂的跳转某行等功能也很难实现。

合理的实现应该能将文本分成很多段，且段与段之间互不影响，这样对其中某些部分进行插入、修改、删除也不会有很大的影响。Zed 选择的是 Rope 这种数据结构。

具体实现里，Rope 是对底层的 SumTree 的一层包装。而 SumTree 可以理解为一种特殊的 B+ 树，显然数据都存在叶子节点，对某些叶子节点的修改并不会影响其他大多数的节点。
通过使用 SumTree，也可以很容易的实现并发访问等功能。

```rust
struct Rope {
    chunks: SumTree<Chunk>,
}

struct Chunk(ArrayString<{ 2 * CHUNK_BASE }>);
```

这里 SumTree 的范型参数为 Chunk，也就是叶子节点存放的数据是 Chunk 类型 (ArrayString 来自 arrayvec 库，是存放在栈里的固定大小的 string)。这里我们首先来看 SumTree 的结构。
```rust
struct SumTree<T: Item>(pub Arc<Node<T>>);

enum Node<T: Item> {
    Internal {
        height: u8,
        summary: T::Summary,
        child_summaries: ArrayVec<T::Summary, { 2 * TREE_BASE }>,
        child_trees: ArrayVec<SumTree<T>, { 2 * TREE_BASE }>,
    },
    Leaf {
        summary: T::Summary,
        items: ArrayVec<T, { 2 * TREE_BASE }>,
        item_summaries: ArrayVec<T::Summary, { 2 * TREE_BASE }>,
    },
}
trait Item: Clone {
    type Summary: Summary;

    fn summary(&self) -> Self::Summary;
}
trait Summary: Default + Clone + fmt::Debug {
    type Context;

    fn add_summary(&mut self, summary: &Self, cx: &Self::Context);
}
```

SumTree<T> 是一棵 B+ 树，其中每个叶节点都包含多个 Item 类型的数据和对应每个 Item 的 Summary， Leaf 的 summary 字段存放着所有 Item 的总 Summary。
Internal 节点包含多个子树和对应每个子树的 Summary， summary 字段存放着所有子树的总 Summary。height 则是该 Internal 节点的高度（叶子节点显然高度为0）。

Item 是什么呢？Item 是存放在叶子节点的数据的真正类型（比如整个文本的一小段子串 &str）。它需要实现 summary 方法返回一个 Summary 类型。
这个 Summary 类型可以是一切能累加的东西。听起来很抽象，但只要知道 Summary 是用来统计整个文本或其中部分文本的一些统计信息的。
比如：如果我想知道整个文本的长度是多少，那我需要知道每个叶子节点的数据长度是多少，最后累加得到最终的结果。
我们可以将这个长度信息通过 Summary 一层一层地传上去不断累加，最终在根节点的 summary 字段即是我们需要的结果。
具体实现上，我们可以给 usize 实现 Summary trait，具体 add_summary 的方法便是直接对两个 usize 相加。

再比如我想知道一个 string 中最大的 char 是哪个？把 add_summary 的方法实现为返回更大的那个 char 即可。

实际中我们会有很多维度的信息需要统计，这时我们可以把这些维度放到一个 struct 中，在 add_summary 方法里对该 struct 的各个字段进行相应的修改即可。
Zed 的 Rope 中的 Summary 是这样的：
```rust
struct TextSummary {
    /// Length in UTF-8
    len: usize,
    /// Length in UTF-16 code units
    len_utf16: OffsetUtf16,
    /// A point representing the number of lines and the length of the last line
    lines: Point,
    /// How many `char`s are in the first line
    first_line_chars: u32,
    /// How many `char`s are in the last line
    last_line_chars: u32,
    /// How many UTF-16 code units are in the last line
    last_line_len_utf16: u32,
    /// The row idx of the longest row
    longest_row: u32,
    /// How many `char`s are in the longest row
    longest_row_chars: u32,
}
```

它是如何实现 Summary trait 的呢：
```rust
impl sum_tree::Summary for TextSummary {
    type Context = ();

    fn add_summary(&mut self, summary: &Self, _: &Self::Context) {
        *self += summary;
    }
}
impl std::ops::Add<Self> for TextSummary {
    type Output = Self;

    fn add(mut self, rhs: Self) -> Self::Output {
        AddAssign::add_assign(&mut self, &rhs);
        self
    }
}

impl<'a> std::ops::AddAssign<&'a Self> for TextSummary {
    fn add_assign(&mut self, other: &'a Self) {
        let joined_chars = self.last_line_chars + other.first_line_chars;
        if joined_chars > self.longest_row_chars {
            self.longest_row = self.lines.row;
            self.longest_row_chars = joined_chars;
        }
        if other.longest_row_chars > self.longest_row_chars {
            self.longest_row = self.lines.row + other.longest_row;
            self.longest_row_chars = other.longest_row_chars;
        }

        if self.lines.row == 0 {
            self.first_line_chars += other.first_line_chars;
        }

        if other.lines.row == 0 {
            self.last_line_chars += other.first_line_chars;
            self.last_line_len_utf16 += other.last_line_len_utf16;
        } else {
            self.last_line_chars = other.last_line_chars;
            self.last_line_len_utf16 = other.last_line_len_utf16;
        }

        self.len += other.len;
        self.len_utf16 += other.len_utf16;
        self.lines += other.lines;
    }
}
```
其中之所以需要 first_line_chars 和 last_line_chars 这两个字段，是为了统计 longest_row_chars，而同一行的内容有可能被存放在前后两个不同的节点中。
