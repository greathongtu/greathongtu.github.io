+++
title = 'Zed 编辑器中的 SumTree 和 Rope 数据结构'
date = 2024-09-06T20:47:19+08:00
draft = false
+++

代码编辑器最重要的组成部分之一就是如何高效地表示和操作文本。最朴素的想法是直接使用 string 数据结构。但作为使用连续内存的数据结构,string 在面对中间位置的数据插入删除时代价过大。并且更复杂的功能,如跳转到某行,也很难高效实现。

一个合理的实现应该能将文本分成很多段,且段与段之间互不影响,这样对其中某些部分进行修改也不会有很大的影响。Zed 编辑器选择的是 Rope 这种数据结构。

## Rope 和 SumTree 概述

在 Zed 的具体实现中,Rope 是对底层 SumTree 的一层包装。而 SumTree 可以理解为一种特殊的 B+ 树,其中数据都存储在叶子节点。这种结构使得对某些叶子节点的修改并不会影响其他大多数的节点。通过使用 SumTree,也可以很容易地实现并发访问等功能。

```rust
struct Rope {
    chunks: SumTree<Chunk>,
}

struct Chunk(ArrayString<{ 2 * CHUNK_BASE }>);
```

这里 SumTree 的范型参数为 Chunk，也就是叶子节点存放的数据是 Chunk 类型 (ArrayString 来自 arrayvec 库，是存放在栈里的固定大小的 string)。这里我们首先来看 SumTree 的结构。

## SumTree 的结构
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

```

SumTree<T> 是一棵 B+ 树,其中:
- 每个叶节点包含多个 Item 类型的数据和对应每个 Item 的 Summary
- Leaf 的 summary 字段存放着所有 Item 的总 Summary
- Internal 节点包含多个子树和对应每个子树的 Summary
- Internal 节点的 summary 字段存放着所有子树的总 Summary
- height 表示该 Internal 节点的高度(叶子节点高度为0)

## Item 和 Summary
Item 是什么呢？Item 是存放在叶子节点的数据的真正类型（比如整个文本的一小段子串 &str）。它需要实现 summary 方法返回一个 Summary 类型。

```rust
trait Item: Clone {
    type Summary: Summary;

    fn summary(&self) -> Self::Summary;
}

trait Summary: Default + Clone + fmt::Debug {
    type Context;

    fn add_summary(&mut self, summary: &Self, cx: &Self::Context);
}
```

Summary 类型可以是一切可累加的(associative）东西。听起来很抽象，但只要知道 Summary 是用来统计整个文本或其中部分文本的一些统计信息的。

比如：如果我想知道整个文本的长度是多少，那我需要知道每个叶子节点的数据长度是多少，最后累加得到最终的结果。
我们可以将这个长度信息通过 Summary 一层一层地传上去不断累加，最终在根节点的 summary 字段即是我们需要的结果。
具体实现上，我们可以给 usize 实现 Summary trait，具体 add_summary 的方法便是直接对两个 usize 相加。

再比如我想知道一个 string 中最大的 char 是哪个？把 add_summary 的方法实现为返回更大的那个 char 即可。

实际中我们会有很多维度的信息需要统计，这时我们可以把这些维度放到一个 struct 中，在 add_summary 方法里对该 struct 的各个字段进行相应的修改即可。

## TextSummary 示例

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


这里举例说明 SumTree 的结构，以下是一段文本：
```text
Hello World!
This is
your captain speaking.
Are you
ready for take-off?
```

根据刚才的文本构建的 SumTree 如下：
![SumTree](/images/sumtree_diagram.png)

## Dimension 和 SeekTarget
有了对 SumTree 大概的理解，我们就可以深入代码研究了。在代码中我们可以发现另两个关键的 trait：Demension 和 SeekTarget。
```rust
/// Each [`Summary`] type can have more than one [`Dimension`] type that it measures.
/// You can use dimensions to seek to a specific location in the [`SumTree`]
/// # Example:
/// Zed's rope has a `TextSummary` type that summarizes lines, characters, and bytes.
/// Each of these are different dimensions we may want to seek to
pub trait Dimension<'a, S: Summary>: Clone + fmt::Debug + Default {
    fn add_summary(&mut self, _summary: &'a S, _: &S::Context);

    fn from_summary(summary: &'a S, cx: &S::Context) -> Self {
        let mut dimension = Self::default();
        dimension.add_summary(summary, cx);
        dimension
    }
}

pub trait SeekTarget<'a, S: Summary, D: Dimension<'a, S>>: fmt::Debug {
    fn cmp(&self, cursor_location: &D, cx: &S::Context) -> Ordering;
}
```

Dimension 是 Summary 的一个维度， SeekTarget 定义了 SumTree 的查找目标。
你可能会问为什么有了 Summary 还需要 Dimension, 这是因为 Summary 存放的是全量的统计信息，而有时候我们只想关注其中的某个维度。
比如在 TextSummary 中，我们可能只关心最长行的字符数而不关心总长度；或者相反。在这种情况下如果我们直接使用 Summary 的 Cursor 遍历会导致性能浪费，
因为在遍历的过程中做了多余的计算，并且过程中累加的 Summary（或者某 Dimension）会被存在 Cursor 的 stack 字段（记录了从根节点到当前叶节点的路径）中，而我们只需要关心其中的一部分信息。


我们可以使用针对某个 Dimension 的 Cursor 来遍历 SumTree，这样就可以得到一个 Dimension 的统计信息而不会有多余的计算。
那么如果我想通过一个 Dimension 的遍历找到某个位置，同时又想得到另一个 Dimension 的信息怎么办呢？
```rust
// Summary is a Dimension
impl<'a, T: Summary> Dimension<'a, T> for T {
    fn add_summary(&mut self, summary: &'a T, cx: &T::Context) {
        Summary::add_summary(self, summary, cx);
    }
}
// Demension is a Target
impl<'a, S: Summary, D: Dimension<'a, S> + Ord> SeekTarget<'a, S, D> for D {
    fn cmp(&self, cursor_location: &Self, _: &S::Context) -> Ordering {
        Ord::cmp(self, cursor_location)
    }
}

impl<'a, T: Summary, D1: Dimension<'a, T>, D2: Dimension<'a, T>> Dimension<'a, T> for (D1, D2) {
    fn add_summary(&mut self, summary: &'a T, cx: &T::Context) {
        self.0.add_summary(summary, cx);
        self.1.add_summary(summary, cx);
    }
}
```

可以看到我们为 (D1, D2) 的 tuple 实现了 Dimension trait，在这个泛型参数为 (D1, D2) 的 Cursor 的遍历过程中我们同时统计了 D1 和 D2 的信息。


简单总结一下，对于一颗 SumTree，他有一种 Item 类型，这个 Item 类型对应一个 Summary 类型。
在使用 Cursor 遍历 SumTree 时，可以通过指定一个 Summary 的子集类型来遍历去聚集，这个子集就是 Dimension。
Cursor 可以通过 SeekTarget 来定位到某个位置，SeekTarget 也可以是一个 Dimension（至少是可以和 Dimension 进行比较）。


## Cursor: SumTree 的遍历器
上面我们提到了 Cursor，这里我们深入研究以下 Cursor 的结构。Cursor 用于遍历 SumTree，它的结构如下：
```rust
#[derive(Clone)]
pub struct Cursor<'a, T: Item, D> {
    // root of SumTree
    tree: &'a SumTree<T>,
    // stack of nodes from root to current node
    stack: ArrayVec<StackEntry<'a, T, D>, 16>,
    // D is what dimension this Cursor is currently seeking
    // position stores the total value of D from the beginning
    position: D,
    did_seek: bool,
    at_end: bool,
}
```
Cursor 允许我们高效地在 SumTree 中导航和查找,同时只计算我们关心的维度的统计信息。

## TODO


## 总结

Zed 编辑器通过使用 Rope 和底层的 SumTree 数据结构，实现了高效的文本表示和操作。
这种设计不仅允许快速的插入、删除和修改操作，还支持复杂的统计和查找功能。
通过 Summary、Dimension 和 SeekTarget 等概念的巧妙运用，Zed 能够灵活地处理各种文本操作需求，
同时保持良好的性能。
