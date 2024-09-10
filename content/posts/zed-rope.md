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

#[derive(Clone)]
struct StackEntry<'a, T: Item, D> {
    // curr node
    tree: &'a SumTree<T>,
    // index of next child to visit
    index: usize,
    // D value of this node
    position: D,
}
```
Cursor 允许我们高效地在 SumTree 中导航和查找,同时只计算我们关心的维度的统计信息。
以下是代码解读：
```rust
// cursor.rs
use super::*;
use arrayvec::ArrayVec;
use std::{cmp::Ordering, mem, sync::Arc};

#[derive(Clone)]
struct StackEntry<'a, T: Item, D> {
    tree: &'a SumTree<T>,
    index: usize,
    position: D,
}

#[derive(Clone)]
pub struct Cursor<'a, T: Item, D> {
    tree: &'a SumTree<T>,
    stack: ArrayVec<StackEntry<'a, T, D>, 16>,
    position: D,
    did_seek: bool,
    at_end: bool,
}

pub struct Iter<'a, T: Item> {
    tree: &'a SumTree<T>,
    stack: ArrayVec<StackEntry<'a, T, ()>, 16>,
}

impl<'a, T, D> Cursor<'a, T, D>
where
    T: Item,
    D: Dimension<'a, T::Summary>,
{
    pub fn new(tree: &'a SumTree<T>) -> Self {
        Self {
            tree,
            stack: ArrayVec::new(),
            position: D::default(),
            did_seek: false,
            at_end: tree.is_empty(),
        }
    }

    fn reset(&mut self) {
        self.did_seek = false;
        self.at_end = self.tree.is_empty();
        self.stack.truncate(0);
        self.position = D::default();
    }

    // position excluding the current item
    pub fn start(&self) -> &D {
        &self.position
    }

    // get total summary until leaf item
    #[track_caller]
    pub fn end(&self, cx: &<T::Summary as Summary>::Context) -> D {
        if let Some(item_summary) = self.item_summary() {
            let mut end = self.start().clone();
            end.add_summary(item_summary, cx);
            end
        } else {
            self.start().clone()
        }
    }

    // item pointed to by curr leaf
    #[track_caller]
    pub fn item(&self) -> Option<&'a T> {
        self.assert_did_seek();
        if let Some(entry) = self.stack.last() {
            match *entry.tree.0 {
                Node::Leaf { ref items, .. } => {
                    if entry.index == items.len() {
                        None
                    } else {
                        Some(&items[entry.index])
                    }
                }
                _ => unreachable!(),
            }
        } else {
            None
        }
    }

    // get the summary of the current item pointed to by curr leaf
    #[track_caller]
    pub fn item_summary(&self) -> Option<&'a T::Summary> {
        self.assert_did_seek();
        if let Some(entry) = self.stack.last() {
            match *entry.tree.0 {
                Node::Leaf {
                    ref item_summaries, ..
                } => {
                    if entry.index == item_summaries.len() {
                        None
                    } else {
                        Some(&item_summaries[entry.index])
                    }
                }
                _ => unreachable!(),
            }
        } else {
            None
        }
    }

    // get next item in the tree
    #[track_caller]
    pub fn next_item(&self) -> Option<&'a T> {
        self.assert_did_seek();
        if let Some(entry) = self.stack.last() {
            if entry.index == entry.tree.0.items().len() - 1 {
                if let Some(next_leaf) = self.next_leaf() {
                    Some(next_leaf.0.items().first().unwrap())
                } else {
                    None
                }
            } else {
                match *entry.tree.0 {
                    Node::Leaf { ref items, .. } => Some(&items[entry.index + 1]),
                    _ => unreachable!(),
                }
            }
        } else if self.at_end {
            None
        } else {
            self.tree.first()
        }
    }

    // get next leaf in the tree
    #[track_caller]
    fn next_leaf(&self) -> Option<&'a SumTree<T>> {
        for entry in self.stack.iter().rev().skip(1) {
            if entry.index < entry.tree.0.child_trees().len() - 1 {
                match *entry.tree.0 {
                    Node::Internal {
                        ref child_trees, ..
                    } => return Some(child_trees[entry.index + 1].leftmost_leaf()),
                    Node::Leaf { .. } => unreachable!(),
                };
            }
        }
        None
    }

    // get prev item in the tree
    #[track_caller]
    pub fn prev_item(&self) -> Option<&'a T> {
        self.assert_did_seek();
        if let Some(entry) = self.stack.last() {
            if entry.index == 0 {
                if let Some(prev_leaf) = self.prev_leaf() {
                    Some(prev_leaf.0.items().last().unwrap())
                } else {
                    None
                }
            } else {
                match *entry.tree.0 {
                    Node::Leaf { ref items, .. } => Some(&items[entry.index - 1]),
                    _ => unreachable!(),
                }
            }
        // when seek_internal finds nothing, we are at the end
        } else if self.at_end {
            self.tree.last()
        } else {
            None
        }
    }

    #[track_caller]
    fn prev_leaf(&self) -> Option<&'a SumTree<T>> {
        for entry in self.stack.iter().rev().skip(1) {
            if entry.index != 0 {
                match *entry.tree.0 {
                    Node::Internal {
                        ref child_trees, ..
                    } => return Some(child_trees[entry.index - 1].rightmost_leaf()),
                    Node::Leaf { .. } => unreachable!(),
                };
            }
        }
        None
    }

    // move Cursor to prev item
    #[track_caller]
    pub fn prev(&mut self, cx: &<T::Summary as Summary>::Context) {
        self.search_backward(|_| true, cx)
    }

    // search backward for first item that satisfies the filter
    #[track_caller]
    pub fn search_backward<F>(&mut self, mut filter_node: F, cx: &<T::Summary as Summary>::Context)
    where
        F: FnMut(&T::Summary) -> bool,
    {
        if !self.did_seek {
            self.did_seek = true;
            self.at_end = true;
        }

        if self.at_end {
            self.position = D::default();
            self.at_end = self.tree.is_empty();
            if !self.tree.is_empty() {
                self.stack.push(StackEntry {
                    tree: self.tree,
                    index: self.tree.0.child_summaries().len(),
                    position: D::from_summary(self.tree.summary(), cx),
                });
            }
        }

        let mut descending = false;
        while !self.stack.is_empty() {
            if let Some(StackEntry { position, .. }) = self.stack.iter().rev().nth(1) {
                self.position = position.clone();
            } else {
                self.position = D::default();
            }

            let entry = self.stack.last_mut().unwrap();
            if !descending {
                if entry.index == 0 {
                    self.stack.pop();
                    continue;
                } else {
                    entry.index -= 1;
                }
            }

            for summary in &entry.tree.0.child_summaries()[..entry.index] {
                self.position.add_summary(summary, cx);
            }
            entry.position = self.position.clone();

            descending = filter_node(&entry.tree.0.child_summaries()[entry.index]);
            match entry.tree.0.as_ref() {
                Node::Internal { child_trees, .. } => {
                    if descending {
                        let tree = &child_trees[entry.index];
                        self.stack.push(StackEntry {
                            position: D::default(),
                            tree,
                            index: tree.0.child_summaries().len() - 1,
                        })
                    }
                }
                Node::Leaf { .. } => {
                    if descending {
                        break;
                    }
                }
            }
        }
    }

    // move Cursor to next item
    #[track_caller]
    pub fn next(&mut self, cx: &<T::Summary as Summary>::Context) {
        self.search_forward(|_| true, cx)
    }

    // search forward for first item that satisfies the filter
    #[track_caller]
    pub fn search_forward<F>(&mut self, mut filter_node: F, cx: &<T::Summary as Summary>::Context)
    where
        F: FnMut(&T::Summary) -> bool,
    {
        let mut descend = false;

        if self.stack.is_empty() {
            if !self.at_end {
                self.stack.push(StackEntry {
                    tree: self.tree,
                    index: 0,
                    position: D::default(),
                });
                descend = true;
            }
            self.did_seek = true;
        }

        while !self.stack.is_empty() {
            let new_subtree = {
                let entry = self.stack.last_mut().unwrap();
                match entry.tree.0.as_ref() {
                    Node::Internal {
                        child_trees,
                        child_summaries,
                        ..
                    } => {
                        if !descend {
                            entry.index += 1;
                            entry.position = self.position.clone();
                        }

                        while entry.index < child_summaries.len() {
                            let next_summary = &child_summaries[entry.index];
                            if filter_node(next_summary) {
                                break;
                            } else {
                                entry.index += 1;
                                entry.position.add_summary(next_summary, cx);
                                self.position.add_summary(next_summary, cx);
                            }
                        }

                        child_trees.get(entry.index)
                    }
                    Node::Leaf { item_summaries, .. } => {
                        if !descend {
                            let item_summary = &item_summaries[entry.index];
                            entry.index += 1;
                            entry.position.add_summary(item_summary, cx);
                            self.position.add_summary(item_summary, cx);
                        }

                        loop {
                            if let Some(next_item_summary) = item_summaries.get(entry.index) {
                                if filter_node(next_item_summary) {
                                    return;
                                } else {
                                    entry.index += 1;
                                    entry.position.add_summary(next_item_summary, cx);
                                    self.position.add_summary(next_item_summary, cx);
                                }
                            } else {
                                break None;
                            }
                        }
                    }
                }
            };

            if let Some(subtree) = new_subtree {
                descend = true;
                self.stack.push(StackEntry {
                    tree: subtree,
                    index: 0,
                    position: self.position.clone(),
                });
            } else {
                descend = false;
                self.stack.pop();
            }
        }

        self.at_end = self.stack.is_empty();
        debug_assert!(self.stack.is_empty() || self.stack.last().unwrap().tree.0.is_leaf());
    }

    #[track_caller]
    fn assert_did_seek(&self) {
        assert!(
            self.did_seek,
            "Must call `seek`, `next` or `prev` before calling this method"
        );
    }
}

impl<'a, T, D> Cursor<'a, T, D>
where
    T: Item,
    D: Dimension<'a, T::Summary>,
{
    #[track_caller]
    pub fn seek<Target>(
        &mut self,
        pos: &Target,
        bias: Bias,
        cx: &<T::Summary as Summary>::Context,
    ) -> bool
    where
        Target: SeekTarget<'a, T::Summary, D>,
    {
        self.reset();
        self.seek_internal(pos, bias, &mut (), cx)
    }

    #[track_caller]
    pub fn seek_forward<Target>(
        &mut self,
        pos: &Target,
        bias: Bias,
        cx: &<T::Summary as Summary>::Context,
    ) -> bool
    where
        Target: SeekTarget<'a, T::Summary, D>,
    {
        self.seek_internal(pos, bias, &mut (), cx)
    }

    // seek to the target position, return a new SumTree that contains the items passed
    #[track_caller]
    pub fn slice<Target>(
        &mut self,
        end: &Target,
        bias: Bias,
        cx: &<T::Summary as Summary>::Context,
    ) -> SumTree<T>
    where
        Target: SeekTarget<'a, T::Summary, D>,
    {
        let mut slice = SliceSeekAggregate {
            tree: SumTree::new(),
            leaf_items: ArrayVec::new(),
            leaf_item_summaries: ArrayVec::new(),
            leaf_summary: T::Summary::default(),
        };
        self.seek_internal(end, bias, &mut slice, cx);
        slice.tree
    }

    // return a new SumTree that contains the items till the end
    #[track_caller]
    pub fn suffix(&mut self, cx: &<T::Summary as Summary>::Context) -> SumTree<T> {
        self.slice(&End::new(), Bias::Right, cx)
    }

    // return Summary from curr to target
    #[track_caller]
    pub fn summary<Target, Output>(
        &mut self,
        end: &Target,
        bias: Bias,
        cx: &<T::Summary as Summary>::Context,
    ) -> Output
    where
        Target: SeekTarget<'a, T::Summary, D>,
        Output: Dimension<'a, T::Summary>,
    {
        let mut summary = SummarySeekAggregate(Output::default());
        self.seek_internal(end, bias, &mut summary, cx);
        summary.0
    }

    /// Returns whether we found the item you where seeking for
    // move Cursor to the last item whose position is less than or equal to the target
    #[track_caller]
    fn seek_internal(
        &mut self,
        target: &dyn SeekTarget<'a, T::Summary, D>,
        bias: Bias,
        aggregate: &mut dyn SeekAggregate<'a, T>,
        cx: &<T::Summary as Summary>::Context,
    ) -> bool {
        debug_assert!(
            target.cmp(&self.position, cx) >= Ordering::Equal,
            "cannot seek backward from {:?} to {:?}",
            self.position,
            target
        );

        if !self.did_seek {
            self.did_seek = true;
            self.stack.push(StackEntry {
                tree: self.tree,
                index: 0,
                position: Default::default(),
            });
        }

        let mut ascending = false;
        'outer: while let Some(entry) = self.stack.last_mut() {
            match *entry.tree.0 {
                Node::Internal {
                    ref child_summaries,
                    ref child_trees,
                    ..
                } => {
                    if ascending {
                        entry.index += 1;
                        entry.position = self.position.clone();
                    }

                    for (child_tree, child_summary) in child_trees[entry.index..]
                        .iter()
                        .zip(&child_summaries[entry.index..])
                    {
                        let mut child_end = self.position.clone();
                        child_end.add_summary(child_summary, cx);

                        let comparison = target.cmp(&child_end, cx);
                        if comparison == Ordering::Greater
                            || (comparison == Ordering::Equal && bias == Bias::Right)
                        {
                            self.position = child_end;
                            aggregate.push_tree(child_tree, child_summary, cx);
                            entry.index += 1;
                            entry.position = self.position.clone();
                        } else {
                            self.stack.push(StackEntry {
                                tree: child_tree,
                                index: 0,
                                position: self.position.clone(),
                            });
                            ascending = false;
                            continue 'outer;
                        }
                    }
                }
                Node::Leaf {
                    ref items,
                    ref item_summaries,
                    ..
                } => {
                    aggregate.begin_leaf();

                    for (item, item_summary) in items[entry.index..]
                        .iter()
                        .zip(&item_summaries[entry.index..])
                    {
                        let mut child_end = self.position.clone();
                        child_end.add_summary(item_summary, cx);

                        let comparison = target.cmp(&child_end, cx);
                        if comparison == Ordering::Greater
                            || (comparison == Ordering::Equal && bias == Bias::Right)
                        {
                            self.position = child_end;
                            aggregate.push_item(item, item_summary, cx);
                            entry.index += 1;
                        } else {
                            aggregate.end_leaf(cx);
                            break 'outer;
                        }
                    }

                    aggregate.end_leaf(cx);
                }
            }

            self.stack.pop();
            ascending = true;
        }

        self.at_end = self.stack.is_empty();
        debug_assert!(self.stack.is_empty() || self.stack.last().unwrap().tree.0.is_leaf());

        let mut end = self.position.clone();
        if bias == Bias::Left {
            if let Some(summary) = self.item_summary() {
                end.add_summary(summary, cx);
            }
        }

        target.cmp(&end, cx) == Ordering::Equal
    }
}

impl<'a, T: Item> Iter<'a, T> {
    pub(crate) fn new(tree: &'a SumTree<T>) -> Self {
        Self {
            tree,
            stack: Default::default(),
        }
    }
}

impl<'a, T: Item> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        let mut descend = false;

        if self.stack.is_empty() {
            self.stack.push(StackEntry {
                tree: self.tree,
                index: 0,
                position: (),
            });
            descend = true;
        }

        while !self.stack.is_empty() {
            let new_subtree = {
                let entry = self.stack.last_mut().unwrap();
                match entry.tree.0.as_ref() {
                    Node::Internal { child_trees, .. } => {
                        if !descend {
                            entry.index += 1;
                        }
                        child_trees.get(entry.index)
                    }
                    Node::Leaf { items, .. } => {
                        if !descend {
                            entry.index += 1;
                        }

                        if let Some(next_item) = items.get(entry.index) {
                            return Some(next_item);
                        } else {
                            None
                        }
                    }
                }
            };

            if let Some(subtree) = new_subtree {
                descend = true;
                self.stack.push(StackEntry {
                    tree: subtree,
                    index: 0,
                    position: (),
                });
            } else {
                descend = false;
                self.stack.pop();
            }
        }

        None
    }
}

impl<'a, T, S, D> Iterator for Cursor<'a, T, D>
where
    T: Item<Summary = S>,
    S: Summary<Context = ()>,
    D: Dimension<'a, T::Summary>,
{
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        // If we haven't already sought to the start of the cursor, do so
        if !self.did_seek {
            self.next(&());
        }

        if let Some(item) = self.item() {
            self.next(&());
            Some(item)
        } else {
            None
        }
    }
}

pub struct FilterCursor<'a, F, T: Item, D> {
    cursor: Cursor<'a, T, D>,
    filter_node: F,
}

impl<'a, F, T, D> FilterCursor<'a, F, T, D>
where
    F: FnMut(&T::Summary) -> bool,
    T: Item,
    D: Dimension<'a, T::Summary>,
{
    pub fn new(tree: &'a SumTree<T>, filter_node: F) -> Self {
        let cursor = tree.cursor::<D>();
        Self {
            cursor,
            filter_node,
        }
    }

    pub fn start(&self) -> &D {
        self.cursor.start()
    }

    pub fn end(&self, cx: &<T::Summary as Summary>::Context) -> D {
        self.cursor.end(cx)
    }

    pub fn item(&self) -> Option<&'a T> {
        self.cursor.item()
    }

    pub fn item_summary(&self) -> Option<&'a T::Summary> {
        self.cursor.item_summary()
    }

    pub fn next(&mut self, cx: &<T::Summary as Summary>::Context) {
        self.cursor.search_forward(&mut self.filter_node, cx);
    }

    pub fn prev(&mut self, cx: &<T::Summary as Summary>::Context) {
        self.cursor.search_backward(&mut self.filter_node, cx);
    }
}

impl<'a, F, T, S, U> Iterator for FilterCursor<'a, F, T, U>
where
    F: FnMut(&T::Summary) -> bool,
    T: Item<Summary = S>,
    S: Summary<Context = ()>, //Context for the summary must be unit type, as .next() doesn't take arguments
    U: Dimension<'a, T::Summary>,
{
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        if !self.cursor.did_seek {
            self.next(&());
        }

        if let Some(item) = self.item() {
            self.cursor.search_forward(&mut self.filter_node, &());
            Some(item)
        } else {
            None
        }
    }
}

trait SeekAggregate<'a, T: Item> {
    fn begin_leaf(&mut self);
    fn end_leaf(&mut self, cx: &<T::Summary as Summary>::Context);
    fn push_item(
        &mut self,
        item: &'a T,
        summary: &'a T::Summary,
        cx: &<T::Summary as Summary>::Context,
    );
    fn push_tree(
        &mut self,
        tree: &'a SumTree<T>,
        summary: &'a T::Summary,
        cx: &<T::Summary as Summary>::Context,
    );
}

// for creating a new tree
struct SliceSeekAggregate<T: Item> {
    tree: SumTree<T>,
    leaf_items: ArrayVec<T, { 2 * TREE_BASE }>,
    leaf_item_summaries: ArrayVec<T::Summary, { 2 * TREE_BASE }>,
    leaf_summary: T::Summary,
}

// for calculating the summary of a tree
struct SummarySeekAggregate<D>(D);

impl<'a, T: Item> SeekAggregate<'a, T> for () {
    fn begin_leaf(&mut self) {}
    fn end_leaf(&mut self, _: &<T::Summary as Summary>::Context) {}
    fn push_item(&mut self, _: &T, _: &T::Summary, _: &<T::Summary as Summary>::Context) {}
    fn push_tree(&mut self, _: &SumTree<T>, _: &T::Summary, _: &<T::Summary as Summary>::Context) {}
}

impl<'a, T: Item> SeekAggregate<'a, T> for SliceSeekAggregate<T> {
    fn begin_leaf(&mut self) {}
    fn end_leaf(&mut self, cx: &<T::Summary as Summary>::Context) {
        self.tree.append(
            SumTree(Arc::new(Node::Leaf {
                summary: mem::take(&mut self.leaf_summary),
                items: mem::take(&mut self.leaf_items),
                item_summaries: mem::take(&mut self.leaf_item_summaries),
            })),
            cx,
        );
    }
    fn push_item(&mut self, item: &T, summary: &T::Summary, cx: &<T::Summary as Summary>::Context) {
        self.leaf_items.push(item.clone());
        self.leaf_item_summaries.push(summary.clone());
        Summary::add_summary(&mut self.leaf_summary, summary, cx);
    }
    fn push_tree(
        &mut self,
        tree: &SumTree<T>,
        _: &T::Summary,
        cx: &<T::Summary as Summary>::Context,
    ) {
        self.tree.append(tree.clone(), cx);
    }
}

impl<'a, T: Item, D> SeekAggregate<'a, T> for SummarySeekAggregate<D>
where
    D: Dimension<'a, T::Summary>,
{
    fn begin_leaf(&mut self) {}
    fn end_leaf(&mut self, _: &<T::Summary as Summary>::Context) {}
    fn push_item(&mut self, _: &T, summary: &'a T::Summary, cx: &<T::Summary as Summary>::Context) {
        self.0.add_summary(summary, cx);
    }
    fn push_tree(
        &mut self,
        _: &SumTree<T>,
        summary: &'a T::Summary,
        cx: &<T::Summary as Summary>::Context,
    ) {
        self.0.add_summary(summary, cx);
    }
}
```

以下是 SumTree 的代码解读：
```rust
// sum_tree.rs
mod cursor;
mod tree_map;

use arrayvec::ArrayVec;
pub use cursor::{Cursor, FilterCursor, Iter};
use rayon::prelude::*;
use std::marker::PhantomData;
use std::mem;
use std::{cmp::Ordering, fmt, iter::FromIterator, sync::Arc};
pub use tree_map::{MapSeekTarget, TreeMap, TreeSet};

#[cfg(test)]
pub const TREE_BASE: usize = 2;
#[cfg(not(test))]
pub const TREE_BASE: usize = 6;

/// An item that can be stored in a [`SumTree`]
///
/// Must be summarized by a type that implements [`Summary`]
pub trait Item: Clone {
    type Summary: Summary;

    fn summary(&self) -> Self::Summary;
}

/// An [`Item`] whose summary has a specific key that can be used to identify it
pub trait KeyedItem: Item {
    type Key: for<'a> Dimension<'a, Self::Summary> + Ord;

    fn key(&self) -> Self::Key;
}

/// A type that describes the Sum of all [`Item`]s in a subtree of the [`SumTree`]
///
/// Each Summary type can have multiple [`Dimensions`] that it measures,
/// which can be used to navigate the tree
pub trait Summary: Default + Clone + fmt::Debug {
    type Context;

    fn add_summary(&mut self, summary: &Self, cx: &Self::Context);
}

/// Each [`Summary`] type can have more than one [`Dimension`] type that it measures.
///
/// You can use dimensions to seek to a specific location in the [`SumTree`]
///
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

impl<'a, T: Summary> Dimension<'a, T> for T {
    fn add_summary(&mut self, summary: &'a T, cx: &T::Context) {
        Summary::add_summary(self, summary, cx);
    }
}

pub trait SeekTarget<'a, S: Summary, D: Dimension<'a, S>>: fmt::Debug {
    fn cmp(&self, cursor_location: &D, cx: &S::Context) -> Ordering;
}

impl<'a, S: Summary, D: Dimension<'a, S> + Ord> SeekTarget<'a, S, D> for D {
    fn cmp(&self, cursor_location: &Self, _: &S::Context) -> Ordering {
        Ord::cmp(self, cursor_location)
    }
}

impl<'a, T: Summary> Dimension<'a, T> for () {
    fn add_summary(&mut self, _: &'a T, _: &T::Context) {}
}

impl<'a, T: Summary, D1: Dimension<'a, T>, D2: Dimension<'a, T>> Dimension<'a, T> for (D1, D2) {
    fn add_summary(&mut self, summary: &'a T, cx: &T::Context) {
        self.0.add_summary(summary, cx);
        self.1.add_summary(summary, cx);
    }
}

impl<'a, S: Summary, D1: SeekTarget<'a, S, D1> + Dimension<'a, S>, D2: Dimension<'a, S>>
    SeekTarget<'a, S, (D1, D2)> for D1
{
    fn cmp(&self, cursor_location: &(D1, D2), cx: &S::Context) -> Ordering {
        self.cmp(&cursor_location.0, cx)
    }
}

struct End<D>(PhantomData<D>);

impl<D> End<D> {
    fn new() -> Self {
        Self(PhantomData)
    }
}

impl<'a, S: Summary, D: Dimension<'a, S>> SeekTarget<'a, S, D> for End<D> {
    fn cmp(&self, _: &D, _: &S::Context) -> Ordering {
        Ordering::Greater
    }
}

impl<D> fmt::Debug for End<D> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_tuple("End").finish()
    }
}

/// Bias is used to settle ambiguities when determining positions in an ordered sequence.
///
/// The primary use case is for text, where Bias influences
/// which character an offset or anchor is associated with.
///
/// # Examples
/// Given the buffer `AˇBCD`:
/// - The offset of the cursor is 1
/// - [Bias::Left] would attach the cursor to the character `A`
/// - [Bias::Right] would attach the cursor to the character `B`
///
/// Given the buffer `A«BCˇ»D`:
/// - The offset of the cursor is 3, and the selection is from 1 to 3
/// - The left anchor of the selection has [Bias::Right], attaching it to the character `B`
/// - The right anchor of the selection has [Bias::Left], attaching it to the character `C`
///
/// Given the buffer `{ˇ<...>`, where `<...>` is a folded region:
/// - The display offset of the cursor is 1, but the offset in the buffer is determined by the bias
/// - [Bias::Left] would attach the cursor to the character `{`, with a buffer offset of 1
/// - [Bias::Right] would attach the cursor to the first character of the folded region,
///   and the buffer offset would be the offset of the first character of the folded region
#[derive(Copy, Clone, Eq, PartialEq, PartialOrd, Ord, Debug, Hash, Default)]
pub enum Bias {
    /// Attach to the character on the left
    #[default]
    Left,
    /// Attach to the character on the right
    Right,
}

impl Bias {
    pub fn invert(self) -> Self {
        match self {
            Self::Left => Self::Right,
            Self::Right => Self::Left,
        }
    }
}

/// A B+ tree in which each leaf node contains `Item`s of type `T` and a `Summary`s for each `Item`.
/// Each internal node contains a `Summary` of the items in its subtree.
///
/// The maximum number of items per node is `TREE_BASE * 2`.
///
/// Any [`Dimension`] supported by the [`Summary`] type can be used to seek to a specific location in the tree.
#[derive(Debug, Clone)]
pub struct SumTree<T: Item>(Arc<Node<T>>);

impl<T: Item> SumTree<T> {
    pub fn new() -> Self {
        SumTree(Arc::new(Node::Leaf {
            summary: T::Summary::default(),
            items: ArrayVec::new(),
            item_summaries: ArrayVec::new(),
        }))
    }

    pub fn from_item(item: T, cx: &<T::Summary as Summary>::Context) -> Self {
        let mut tree = Self::new();
        tree.push(item, cx);
        tree
    }

    // return SumTree from iterator of items. Every 2*TREE_BASE items collect into a leaf, every 2*TREE_BASE nodes collect into a parent node
    pub fn from_iter<I: IntoIterator<Item = T>>(
        iter: I,
        cx: &<T::Summary as Summary>::Context,
    ) -> Self {
        let mut nodes = Vec::new();

        let mut iter = iter.into_iter().fuse().peekable();
        while iter.peek().is_some() {
            let items: ArrayVec<T, { 2 * TREE_BASE }> = iter.by_ref().take(2 * TREE_BASE).collect();
            let item_summaries: ArrayVec<T::Summary, { 2 * TREE_BASE }> =
                items.iter().map(|item| item.summary()).collect();

            let mut summary = item_summaries[0].clone();
            for item_summary in &item_summaries[1..] {
                <T::Summary as Summary>::add_summary(&mut summary, item_summary, cx);
            }

            nodes.push(Node::Leaf {
                summary,
                items,
                item_summaries,
            });
        }

        let mut parent_nodes = Vec::new();
        let mut height = 0;
        while nodes.len() > 1 {
            height += 1;
            let mut current_parent_node = None;
            for child_node in nodes.drain(..) {
                let parent_node = current_parent_node.get_or_insert_with(|| Node::Internal {
                    summary: T::Summary::default(),
                    height,
                    child_summaries: ArrayVec::new(),
                    child_trees: ArrayVec::new(),
                });
                let Node::Internal {
                    summary,
                    child_summaries,
                    child_trees,
                    ..
                } = parent_node
                else {
                    unreachable!()
                };
                let child_summary = child_node.summary();
                <T::Summary as Summary>::add_summary(summary, child_summary, cx);
                child_summaries.push(child_summary.clone());
                child_trees.push(Self(Arc::new(child_node)));

                if child_trees.len() == 2 * TREE_BASE {
                    parent_nodes.extend(current_parent_node.take());
                }
            }
            parent_nodes.extend(current_parent_node.take());
            mem::swap(&mut nodes, &mut parent_nodes);
        }

        if nodes.is_empty() {
            Self::new()
        } else {
            debug_assert_eq!(nodes.len(), 1);
            Self(Arc::new(nodes.pop().unwrap()))
        }
    }

    // use rayon to create SumTree parallelly
    pub fn from_par_iter<I, Iter>(iter: I, cx: &<T::Summary as Summary>::Context) -> Self
    where
        I: IntoParallelIterator<Iter = Iter>,
        Iter: IndexedParallelIterator<Item = T>,
        T: Send + Sync,
        T::Summary: Send + Sync,
        <T::Summary as Summary>::Context: Sync,
    {
        let mut nodes = iter
            .into_par_iter()
            .chunks(2 * TREE_BASE)
            .map(|items| {
                let items: ArrayVec<T, { 2 * TREE_BASE }> = items.into_iter().collect();
                let item_summaries: ArrayVec<T::Summary, { 2 * TREE_BASE }> =
                    items.iter().map(|item| item.summary()).collect();
                let mut summary = item_summaries[0].clone();
                for item_summary in &item_summaries[1..] {
                    <T::Summary as Summary>::add_summary(&mut summary, item_summary, cx);
                }
                SumTree(Arc::new(Node::Leaf {
                    summary,
                    items,
                    item_summaries,
                }))
            })
            .collect::<Vec<_>>();

        let mut height = 0;
        while nodes.len() > 1 {
            height += 1;
            nodes = nodes
                .into_par_iter()
                .chunks(2 * TREE_BASE)
                .map(|child_nodes| {
                    let child_trees: ArrayVec<SumTree<T>, { 2 * TREE_BASE }> =
                        child_nodes.into_iter().collect();
                    let child_summaries: ArrayVec<T::Summary, { 2 * TREE_BASE }> = child_trees
                        .iter()
                        .map(|child_tree| child_tree.summary().clone())
                        .collect();
                    let mut summary = child_summaries[0].clone();
                    for child_summary in &child_summaries[1..] {
                        <T::Summary as Summary>::add_summary(&mut summary, child_summary, cx);
                    }
                    SumTree(Arc::new(Node::Internal {
                        height,
                        summary,
                        child_summaries,
                        child_trees,
                    }))
                })
                .collect::<Vec<_>>();
        }

        if nodes.is_empty() {
            Self::new()
        } else {
            debug_assert_eq!(nodes.len(), 1);
            nodes.pop().unwrap()
        }
    }

    // get all items of the whole tree
    #[allow(unused)]
    pub fn items(&self, cx: &<T::Summary as Summary>::Context) -> Vec<T> {
        let mut items = Vec::new();
        let mut cursor = self.cursor::<()>();
        cursor.next(cx);
        while let Some(item) = cursor.item() {
            items.push(item.clone());
            cursor.next(cx);
        }
        items
    }

    pub fn iter(&self) -> Iter<T> {
        Iter::new(self)
    }

    pub fn cursor<'a, S>(&'a self) -> Cursor<T, S>
    where
        S: Dimension<'a, T::Summary>,
    {
        Cursor::new(self)
    }

    /// Note: If the summary type requires a non `()` context, then the filter cursor
    /// that is returned cannot be used with Rust's iterators.
    pub fn filter<'a, F, U>(&'a self, filter_node: F) -> FilterCursor<F, T, U>
    where
        F: FnMut(&T::Summary) -> bool,
        U: Dimension<'a, T::Summary>,
    {
        FilterCursor::new(self, filter_node)
    }

    // return first item
    #[allow(dead_code)]
    pub fn first(&self) -> Option<&T> {
        self.leftmost_leaf().0.items().first()
    }

    // return last item
    pub fn last(&self) -> Option<&T> {
        self.rightmost_leaf().0.items().last()
    }

    // update last item
    pub fn update_last(&mut self, f: impl FnOnce(&mut T), cx: &<T::Summary as Summary>::Context) {
        self.update_last_recursive(f, cx);
    }

    // update last item and return whole whole Summary
    fn update_last_recursive(
        &mut self,
        f: impl FnOnce(&mut T),
        cx: &<T::Summary as Summary>::Context,
    ) -> Option<T::Summary> {
        match Arc::make_mut(&mut self.0) {
            Node::Internal {
                summary,
                child_summaries,
                child_trees,
                ..
            } => {
                let last_summary = child_summaries.last_mut().unwrap();
                let last_child = child_trees.last_mut().unwrap();
                *last_summary = last_child.update_last_recursive(f, cx).unwrap();
                *summary = sum(child_summaries.iter(), cx);
                Some(summary.clone())
            }
            Node::Leaf {
                summary,
                items,
                item_summaries,
            } => {
                if let Some((item, item_summary)) = items.last_mut().zip(item_summaries.last_mut())
                {
                    (f)(item);
                    *item_summary = item.summary();
                    *summary = sum(item_summaries.iter(), cx);
                    Some(summary.clone())
                } else {
                    None
                }
            }
        }
    }

    // return whole Dimension of the tree
    pub fn extent<'a, D: Dimension<'a, T::Summary>>(
        &'a self,
        cx: &<T::Summary as Summary>::Context,
    ) -> D {
        let mut extent = D::default();
        match self.0.as_ref() {
            Node::Internal { summary, .. } | Node::Leaf { summary, .. } => {
                extent.add_summary(summary, cx);
            }
        }
        extent
    }

    // return whole Summary of the tree
    pub fn summary(&self) -> &T::Summary {
        match self.0.as_ref() {
            Node::Internal { summary, .. } => summary,
            Node::Leaf { summary, .. } => summary,
        }
    }

    // whether the SumTree is empty
    pub fn is_empty(&self) -> bool {
        match self.0.as_ref() {
            Node::Internal { .. } => false,
            Node::Leaf { items, .. } => items.is_empty(),
        }
    }

    // change iterator of Item to a new SumTree, then merge two Trees
    pub fn extend<I>(&mut self, iter: I, cx: &<T::Summary as Summary>::Context)
    where
        I: IntoIterator<Item = T>,
    {
        self.append(Self::from_iter(iter, cx), cx);
    }

    // change parallel iterator of Item to a new SumTree, then merge two Trees
    pub fn par_extend<I, Iter>(&mut self, iter: I, cx: &<T::Summary as Summary>::Context)
    where
        I: IntoParallelIterator<Iter = Iter>,
        Iter: IndexedParallelIterator<Item = T>,
        T: Send + Sync,
        T::Summary: Send + Sync,
        <T::Summary as Summary>::Context: Sync,
    {
        self.append(Self::from_par_iter(iter, cx), cx);
    }

    // push one item into SumTree
    pub fn push(&mut self, item: T, cx: &<T::Summary as Summary>::Context) {
        let summary = item.summary();
        self.append(
            SumTree(Arc::new(Node::Leaf {
                summary: summary.clone(),
                items: ArrayVec::from_iter(Some(item)),
                item_summaries: ArrayVec::from_iter(Some(summary)),
            })),
            cx,
        );
    }

    // append other tree into self tree
    pub fn append(&mut self, other: Self, cx: &<T::Summary as Summary>::Context) {
        if self.is_empty() {
            *self = other;
        } else if !other.0.is_leaf() || !other.0.items().is_empty() {
            if self.0.height() < other.0.height() {
                for tree in other.0.child_trees() {
                    self.append(tree.clone(), cx);
                }
            } else if let Some(split_tree) = self.push_tree_recursive(other, cx) {
                *self = Self::from_child_trees(self.clone(), split_tree, cx);
            }
        }
    }

    // self height must >= other height
    // merge two SumTree, return potentially splited new tree for next step from_child_trees function call
    fn push_tree_recursive(
        &mut self,
        other: SumTree<T>,
        cx: &<T::Summary as Summary>::Context,
    ) -> Option<SumTree<T>> {
        match Arc::make_mut(&mut self.0) {
            Node::Internal {
                height,
                summary,
                child_summaries,
                child_trees,
                ..
            } => {
                let other_node = other.0.clone();
                <T::Summary as Summary>::add_summary(summary, other_node.summary(), cx);

                let height_delta = *height - other_node.height();
                let mut summaries_to_append = ArrayVec::<T::Summary, { 2 * TREE_BASE }>::new();
                let mut trees_to_append = ArrayVec::<SumTree<T>, { 2 * TREE_BASE }>::new();
                // Based on the height difference, it decides how to append the other tree:
                //  If heights are equal, it appends all child summaries and trees from the other node.
                //  If the height difference is 1 and the other node isn't underflowing, it appends the other tree as a whole.
                //  Otherwise, it recursively calls push_tree_recursive on the last child of the current node.
                // If the number of children exceeds the maximum allowed (2 TREE_BASE), it splits the node:
                //  It divides the children into two groups.
                //  It updates the current node with the left group.
                //  It returns a new splited SumTree with the right group as Some(SumTree).
                if height_delta == 0 {
                    summaries_to_append.extend(other_node.child_summaries().iter().cloned());
                    trees_to_append.extend(other_node.child_trees().iter().cloned());
                } else if height_delta == 1 && !other_node.is_underflowing() {
                    summaries_to_append.push(other_node.summary().clone());
                    trees_to_append.push(other)
                } else {
                    let tree_to_append = child_trees
                        .last_mut()
                        .unwrap()
                        .push_tree_recursive(other, cx);
                    *child_summaries.last_mut().unwrap() =
                        child_trees.last().unwrap().0.summary().clone();

                    if let Some(split_tree) = tree_to_append {
                        summaries_to_append.push(split_tree.0.summary().clone());
                        trees_to_append.push(split_tree);
                    }
                }

                let child_count = child_trees.len() + trees_to_append.len();
                if child_count > 2 * TREE_BASE {
                    let left_summaries: ArrayVec<_, { 2 * TREE_BASE }>;
                    let right_summaries: ArrayVec<_, { 2 * TREE_BASE }>;
                    let left_trees;
                    let right_trees;

                    let midpoint = (child_count + child_count % 2) / 2;
                    {
                        let mut all_summaries = child_summaries
                            .iter()
                            .chain(summaries_to_append.iter())
                            .cloned();
                        left_summaries = all_summaries.by_ref().take(midpoint).collect();
                        right_summaries = all_summaries.collect();
                        let mut all_trees =
                            child_trees.iter().chain(trees_to_append.iter()).cloned();
                        left_trees = all_trees.by_ref().take(midpoint).collect();
                        right_trees = all_trees.collect();
                    }
                    *summary = sum(left_summaries.iter(), cx);
                    *child_summaries = left_summaries;
                    *child_trees = left_trees;

                    Some(SumTree(Arc::new(Node::Internal {
                        height: *height,
                        summary: sum(right_summaries.iter(), cx),
                        child_summaries: right_summaries,
                        child_trees: right_trees,
                    })))
                } else {
                    child_summaries.extend(summaries_to_append);
                    child_trees.extend(trees_to_append);
                    None
                }
            }
            Node::Leaf {
                summary,
                items,
                item_summaries,
            } => {
                let other_node = other.0;

                // If the combined number of items exceeds the maximum allowed, it splits the leaf:
                // It divides the items into two groups. current leaf is left group. returns a new SumTree with the right group as Some(SumTree).
                let child_count = items.len() + other_node.items().len();
                if child_count > 2 * TREE_BASE {
                    let left_items;
                    let right_items;
                    let left_summaries;
                    let right_summaries: ArrayVec<T::Summary, { 2 * TREE_BASE }>;

                    let midpoint = (child_count + child_count % 2) / 2;
                    {
                        let mut all_items = items.iter().chain(other_node.items().iter()).cloned();
                        left_items = all_items.by_ref().take(midpoint).collect();
                        right_items = all_items.collect();

                        let mut all_summaries = item_summaries
                            .iter()
                            .chain(other_node.child_summaries())
                            .cloned();
                        left_summaries = all_summaries.by_ref().take(midpoint).collect();
                        right_summaries = all_summaries.collect();
                    }
                    *items = left_items;
                    *item_summaries = left_summaries;
                    *summary = sum(item_summaries.iter(), cx);
                    Some(SumTree(Arc::new(Node::Leaf {
                        items: right_items,
                        summary: sum(right_summaries.iter(), cx),
                        item_summaries: right_summaries,
                    })))
                } else {
                    <T::Summary as Summary>::add_summary(summary, other_node.summary(), cx);
                    items.extend(other_node.items().iter().cloned());
                    item_summaries.extend(other_node.child_summaries().iter().cloned());
                    None
                }
            }
        }
    }

    // merge two child trees into a parent tree
    fn from_child_trees(
        left: SumTree<T>,
        right: SumTree<T>,
        cx: &<T::Summary as Summary>::Context,
    ) -> Self {
        let height = left.0.height() + 1;
        let mut child_summaries = ArrayVec::new();
        child_summaries.push(left.0.summary().clone());
        child_summaries.push(right.0.summary().clone());
        let mut child_trees = ArrayVec::new();
        child_trees.push(left);
        child_trees.push(right);
        SumTree(Arc::new(Node::Internal {
            height,
            summary: sum(child_summaries.iter(), cx),
            child_summaries,
            child_trees,
        }))
    }

    // return leftmost leaf SumTree
    fn leftmost_leaf(&self) -> &Self {
        match *self.0 {
            Node::Leaf { .. } => self,
            Node::Internal {
                ref child_trees, ..
            } => child_trees.first().unwrap().leftmost_leaf(),
        }
    }

    // return rightmost leaf SumTree
    fn rightmost_leaf(&self) -> &Self {
        match *self.0 {
            Node::Leaf { .. } => self,
            Node::Internal {
                ref child_trees, ..
            } => child_trees.last().unwrap().rightmost_leaf(),
        }
    }

    #[cfg(debug_assertions)]
    pub fn _debug_entries(&self) -> Vec<&T> {
        self.iter().collect::<Vec<_>>()
    }
}

impl<T: Item + PartialEq> PartialEq for SumTree<T> {
    fn eq(&self, other: &Self) -> bool {
        self.iter().eq(other.iter())
    }
}

impl<T: Item + Eq> Eq for SumTree<T> {}

impl<T: KeyedItem> SumTree<T> {
    // insert or replace item
    pub fn insert_or_replace(
        &mut self,
        item: T,
        cx: &<T::Summary as Summary>::Context,
    ) -> Option<T> {
        let mut replaced = None;
        *self = {
            let mut cursor = self.cursor::<T::Key>();
            let mut new_tree = cursor.slice(&item.key(), Bias::Left, cx);
            if let Some(cursor_item) = cursor.item() {
                if cursor_item.key() == item.key() {
                    replaced = Some(cursor_item.clone());
                    cursor.next(cx);
                }
            }
            new_tree.push(item, cx);
            new_tree.append(cursor.suffix(cx), cx);
            new_tree
        };
        replaced
    }

    // remove item
    pub fn remove(&mut self, key: &T::Key, cx: &<T::Summary as Summary>::Context) -> Option<T> {
        let mut removed = None;
        *self = {
            let mut cursor = self.cursor::<T::Key>();
            let mut new_tree = cursor.slice(key, Bias::Left, cx);
            if let Some(item) = cursor.item() {
                if item.key() == *key {
                    removed = Some(item.clone());
                    cursor.next(cx);
                }
            }
            new_tree.append(cursor.suffix(cx), cx);
            new_tree
        };
        removed
    }

    // edit SumTree with order for efficiency. return removed items
    pub fn edit(
        &mut self,
        mut edits: Vec<Edit<T>>,
        cx: &<T::Summary as Summary>::Context,
    ) -> Vec<T> {
        if edits.is_empty() {
            return Vec::new();
        }

        let mut removed = Vec::new();
        edits.sort_unstable_by_key(|item| item.key());

        *self = {
            let mut cursor = self.cursor::<T::Key>();
            let mut new_tree = SumTree::new();
            let mut buffered_items = Vec::new();

            cursor.seek(&T::Key::default(), Bias::Left, cx);
            for edit in edits {
                let new_key = edit.key();
                let mut old_item = cursor.item();

                if old_item
                    .as_ref()
                    .map_or(false, |old_item| old_item.key() < new_key)
                {
                    new_tree.extend(buffered_items.drain(..), cx);
                    let slice = cursor.slice(&new_key, Bias::Left, cx);
                    new_tree.append(slice, cx);
                    old_item = cursor.item();
                }

                if let Some(old_item) = old_item {
                    if old_item.key() == new_key {
                        removed.push(old_item.clone());
                        cursor.next(cx);
                    }
                }

                match edit {
                    Edit::Insert(item) => {
                        buffered_items.push(item);
                    }
                    Edit::Remove(_) => {}
                }
            }

            new_tree.extend(buffered_items, cx);
            new_tree.append(cursor.suffix(cx), cx);
            new_tree
        };

        removed
    }

    // get item of key
    pub fn get(&self, key: &T::Key, cx: &<T::Summary as Summary>::Context) -> Option<&T> {
        let mut cursor = self.cursor::<T::Key>();
        if cursor.seek(key, Bias::Left, cx) {
            cursor.item()
        } else {
            None
        }
    }
}

impl<T: Item> Default for SumTree<T> {
    fn default() -> Self {
        Self::new()
    }
}

#[derive(Clone, Debug)]
pub enum Node<T: Item> {
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

impl<T: Item> Node<T> {
    fn is_leaf(&self) -> bool {
        matches!(self, Node::Leaf { .. })
    }

    fn height(&self) -> u8 {
        match self {
            Node::Internal { height, .. } => *height,
            Node::Leaf { .. } => 0,
        }
    }

    fn summary(&self) -> &T::Summary {
        match self {
            Node::Internal { summary, .. } => summary,
            Node::Leaf { summary, .. } => summary,
        }
    }

    fn child_summaries(&self) -> &[T::Summary] {
        match self {
            Node::Internal {
                child_summaries, ..
            } => child_summaries.as_slice(),
            Node::Leaf { item_summaries, .. } => item_summaries.as_slice(),
        }
    }

    // return child trees of curr node
    fn child_trees(&self) -> &ArrayVec<SumTree<T>, { 2 * TREE_BASE }> {
        match self {
            Node::Internal { child_trees, .. } => child_trees,
            Node::Leaf { .. } => panic!("Leaf nodes have no child trees"),
        }
    }

    // return items of curr leaf node
    fn items(&self) -> &ArrayVec<T, { 2 * TREE_BASE }> {
        match self {
            Node::Leaf { items, .. } => items,
            Node::Internal { .. } => panic!("Internal nodes have no items"),
        }
    }

    // whether child num is less than TREE_BASE
    fn is_underflowing(&self) -> bool {
        match self {
            Node::Internal { child_trees, .. } => child_trees.len() < TREE_BASE,
            Node::Leaf { items, .. } => items.len() < TREE_BASE,
        }
    }
}

#[derive(Debug)]
pub enum Edit<T: KeyedItem> {
    Insert(T),
    Remove(T::Key),
}

impl<T: KeyedItem> Edit<T> {
    fn key(&self) -> T::Key {
        match self {
            Edit::Insert(item) => item.key(),
            Edit::Remove(key) => key.clone(),
        }
    }
}

// return whole Summary of the item iterator
fn sum<'a, T, I>(iter: I, cx: &T::Context) -> T
where
    T: 'a + Summary,
    I: Iterator<Item = &'a T>,
{
    let mut sum = T::default();
    for value in iter {
        sum.add_summary(value, cx);
    }
    sum
}

#[cfg(test)]
mod tests {
    use super::*;
    use rand::{distributions, prelude::*};
    use std::cmp;

    #[ctor::ctor]
    fn init_logger() {
        if std::env::var("RUST_LOG").is_ok() {
            env_logger::init();
        }
    }

    #[test]
    fn test_extend_and_push_tree() {
        let mut tree1 = SumTree::new();
        tree1.extend(0..20, &());

        let mut tree2 = SumTree::new();
        tree2.extend(50..100, &());

        tree1.append(tree2, &());
        assert_eq!(
            tree1.items(&()),
            (0..20).chain(50..100).collect::<Vec<u8>>()
        );
    }

    #[test]
    fn test_random() {
        let mut starting_seed = 0;
        if let Ok(value) = std::env::var("SEED") {
            starting_seed = value.parse().expect("invalid SEED variable");
        }
        let mut num_iterations = 100;
        if let Ok(value) = std::env::var("ITERATIONS") {
            num_iterations = value.parse().expect("invalid ITERATIONS variable");
        }
        let num_operations = std::env::var("OPERATIONS")
            .map_or(5, |o| o.parse().expect("invalid OPERATIONS variable"));

        for seed in starting_seed..(starting_seed + num_iterations) {
            eprintln!("seed = {}", seed);
            let mut rng = StdRng::seed_from_u64(seed);

            let rng = &mut rng;
            let mut tree = SumTree::<u8>::new();
            let count = rng.gen_range(0..10);
            if rng.gen() {
                tree.extend(rng.sample_iter(distributions::Standard).take(count), &());
            } else {
                let items = rng
                    .sample_iter(distributions::Standard)
                    .take(count)
                    .collect::<Vec<_>>();
                tree.par_extend(items, &());
            }

            for _ in 0..num_operations {
                let splice_end = rng.gen_range(0..tree.extent::<Count>(&()).0 + 1);
                let splice_start = rng.gen_range(0..splice_end + 1);
                let count = rng.gen_range(0..10);
                let tree_end = tree.extent::<Count>(&());
                let new_items = rng
                    .sample_iter(distributions::Standard)
                    .take(count)
                    .collect::<Vec<u8>>();

                let mut reference_items = tree.items(&());
                reference_items.splice(splice_start..splice_end, new_items.clone());

                tree = {
                    let mut cursor = tree.cursor::<Count>();
                    let mut new_tree = cursor.slice(&Count(splice_start), Bias::Right, &());
                    if rng.gen() {
                        new_tree.extend(new_items, &());
                    } else {
                        new_tree.par_extend(new_items, &());
                    }
                    cursor.seek(&Count(splice_end), Bias::Right, &());
                    new_tree.append(cursor.slice(&tree_end, Bias::Right, &()), &());
                    new_tree
                };

                assert_eq!(tree.items(&()), reference_items);
                assert_eq!(
                    tree.iter().collect::<Vec<_>>(),
                    tree.cursor::<()>().collect::<Vec<_>>()
                );

                log::info!("tree items: {:?}", tree.items(&()));

                let mut filter_cursor = tree.filter::<_, Count>(|summary| summary.contains_even);
                let expected_filtered_items = tree
                    .items(&())
                    .into_iter()
                    .enumerate()
                    .filter(|(_, item)| (item & 1) == 0)
                    .collect::<Vec<_>>();

                let mut item_ix = if rng.gen() {
                    filter_cursor.next(&());
                    0
                } else {
                    filter_cursor.prev(&());
                    expected_filtered_items.len().saturating_sub(1)
                };
                while item_ix < expected_filtered_items.len() {
                    log::info!("filter_cursor, item_ix: {}", item_ix);
                    let actual_item = filter_cursor.item().unwrap();
                    let (reference_index, reference_item) = expected_filtered_items[item_ix];
                    assert_eq!(actual_item, &reference_item);
                    assert_eq!(filter_cursor.start().0, reference_index);
                    log::info!("next");
                    filter_cursor.next(&());
                    item_ix += 1;

                    while item_ix > 0 && rng.gen_bool(0.2) {
                        log::info!("prev");
                        filter_cursor.prev(&());
                        item_ix -= 1;

                        if item_ix == 0 && rng.gen_bool(0.2) {
                            filter_cursor.prev(&());
                            assert_eq!(filter_cursor.item(), None);
                            assert_eq!(filter_cursor.start().0, 0);
                            filter_cursor.next(&());
                        }
                    }
                }
                assert_eq!(filter_cursor.item(), None);

                let mut before_start = false;
                let mut cursor = tree.cursor::<Count>();
                let start_pos = rng.gen_range(0..=reference_items.len());
                cursor.seek(&Count(start_pos), Bias::Right, &());
                let mut pos = rng.gen_range(start_pos..=reference_items.len());
                cursor.seek_forward(&Count(pos), Bias::Right, &());

                for i in 0..10 {
                    assert_eq!(cursor.start().0, pos);

                    if pos > 0 {
                        assert_eq!(cursor.prev_item().unwrap(), &reference_items[pos - 1]);
                    } else {
                        assert_eq!(cursor.prev_item(), None);
                    }

                    if pos < reference_items.len() && !before_start {
                        assert_eq!(cursor.item().unwrap(), &reference_items[pos]);
                    } else {
                        assert_eq!(cursor.item(), None);
                    }

                    if before_start {
                        assert_eq!(cursor.next_item(), reference_items.get(0));
                    } else if pos + 1 < reference_items.len() {
                        assert_eq!(cursor.next_item().unwrap(), &reference_items[pos + 1]);
                    } else {
                        assert_eq!(cursor.next_item(), None);
                    }

                    if i < 5 {
                        cursor.next(&());
                        if pos < reference_items.len() {
                            pos += 1;
                            before_start = false;
                        }
                    } else {
                        cursor.prev(&());
                        if pos == 0 {
                            before_start = true;
                        }
                        pos = pos.saturating_sub(1);
                    }
                }
            }

            for _ in 0..10 {
                let end = rng.gen_range(0..tree.extent::<Count>(&()).0 + 1);
                let start = rng.gen_range(0..end + 1);
                let start_bias = if rng.gen() { Bias::Left } else { Bias::Right };
                let end_bias = if rng.gen() { Bias::Left } else { Bias::Right };

                let mut cursor = tree.cursor::<Count>();
                cursor.seek(&Count(start), start_bias, &());
                let slice = cursor.slice(&Count(end), end_bias, &());

                cursor.seek(&Count(start), start_bias, &());
                let summary = cursor.summary::<_, Sum>(&Count(end), end_bias, &());

                assert_eq!(summary.0, slice.summary().sum);
            }
        }
    }

    #[test]
    fn test_cursor() {
        // Empty tree
        let tree = SumTree::<u8>::new();
        let mut cursor = tree.cursor::<IntegersSummary>();
        assert_eq!(
            cursor.slice(&Count(0), Bias::Right, &()).items(&()),
            Vec::<u8>::new()
        );
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), None);
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 0);
        cursor.prev(&());
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), None);
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 0);
        cursor.next(&());
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), None);
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 0);

        // Single-element tree
        let mut tree = SumTree::<u8>::new();
        tree.extend(vec![1], &());
        let mut cursor = tree.cursor::<IntegersSummary>();
        assert_eq!(
            cursor.slice(&Count(0), Bias::Right, &()).items(&()),
            Vec::<u8>::new()
        );
        assert_eq!(cursor.item(), Some(&1));
        assert_eq!(cursor.prev_item(), None);
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 0);

        cursor.next(&());
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), Some(&1));
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 1);

        cursor.prev(&());
        assert_eq!(cursor.item(), Some(&1));
        assert_eq!(cursor.prev_item(), None);
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 0);

        let mut cursor = tree.cursor::<IntegersSummary>();
        assert_eq!(cursor.slice(&Count(1), Bias::Right, &()).items(&()), [1]);
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), Some(&1));
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 1);

        cursor.seek(&Count(0), Bias::Right, &());
        assert_eq!(
            cursor
                .slice(&tree.extent::<Count>(&()), Bias::Right, &())
                .items(&()),
            [1]
        );
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), Some(&1));
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 1);

        // Multiple-element tree
        let mut tree = SumTree::new();
        tree.extend(vec![1, 2, 3, 4, 5, 6], &());
        let mut cursor = tree.cursor::<IntegersSummary>();

        assert_eq!(cursor.slice(&Count(2), Bias::Right, &()).items(&()), [1, 2]);
        assert_eq!(cursor.item(), Some(&3));
        assert_eq!(cursor.prev_item(), Some(&2));
        assert_eq!(cursor.next_item(), Some(&4));
        assert_eq!(cursor.start().sum, 3);

        cursor.next(&());
        assert_eq!(cursor.item(), Some(&4));
        assert_eq!(cursor.prev_item(), Some(&3));
        assert_eq!(cursor.next_item(), Some(&5));
        assert_eq!(cursor.start().sum, 6);

        cursor.next(&());
        assert_eq!(cursor.item(), Some(&5));
        assert_eq!(cursor.prev_item(), Some(&4));
        assert_eq!(cursor.next_item(), Some(&6));
        assert_eq!(cursor.start().sum, 10);

        cursor.next(&());
        assert_eq!(cursor.item(), Some(&6));
        assert_eq!(cursor.prev_item(), Some(&5));
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 15);

        cursor.next(&());
        cursor.next(&());
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), Some(&6));
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 21);

        cursor.prev(&());
        assert_eq!(cursor.item(), Some(&6));
        assert_eq!(cursor.prev_item(), Some(&5));
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 15);

        cursor.prev(&());
        assert_eq!(cursor.item(), Some(&5));
        assert_eq!(cursor.prev_item(), Some(&4));
        assert_eq!(cursor.next_item(), Some(&6));
        assert_eq!(cursor.start().sum, 10);

        cursor.prev(&());
        assert_eq!(cursor.item(), Some(&4));
        assert_eq!(cursor.prev_item(), Some(&3));
        assert_eq!(cursor.next_item(), Some(&5));
        assert_eq!(cursor.start().sum, 6);

        cursor.prev(&());
        assert_eq!(cursor.item(), Some(&3));
        assert_eq!(cursor.prev_item(), Some(&2));
        assert_eq!(cursor.next_item(), Some(&4));
        assert_eq!(cursor.start().sum, 3);

        cursor.prev(&());
        assert_eq!(cursor.item(), Some(&2));
        assert_eq!(cursor.prev_item(), Some(&1));
        assert_eq!(cursor.next_item(), Some(&3));
        assert_eq!(cursor.start().sum, 1);

        cursor.prev(&());
        assert_eq!(cursor.item(), Some(&1));
        assert_eq!(cursor.prev_item(), None);
        assert_eq!(cursor.next_item(), Some(&2));
        assert_eq!(cursor.start().sum, 0);

        cursor.prev(&());
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), None);
        assert_eq!(cursor.next_item(), Some(&1));
        assert_eq!(cursor.start().sum, 0);

        cursor.next(&());
        assert_eq!(cursor.item(), Some(&1));
        assert_eq!(cursor.prev_item(), None);
        assert_eq!(cursor.next_item(), Some(&2));
        assert_eq!(cursor.start().sum, 0);

        let mut cursor = tree.cursor::<IntegersSummary>();
        assert_eq!(
            cursor
                .slice(&tree.extent::<Count>(&()), Bias::Right, &())
                .items(&()),
            tree.items(&())
        );
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), Some(&6));
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 21);

        cursor.seek(&Count(3), Bias::Right, &());
        assert_eq!(
            cursor
                .slice(&tree.extent::<Count>(&()), Bias::Right, &())
                .items(&()),
            [4, 5, 6]
        );
        assert_eq!(cursor.item(), None);
        assert_eq!(cursor.prev_item(), Some(&6));
        assert_eq!(cursor.next_item(), None);
        assert_eq!(cursor.start().sum, 21);

        // Seeking can bias left or right
        cursor.seek(&Count(1), Bias::Left, &());
        assert_eq!(cursor.item(), Some(&1));
        cursor.seek(&Count(1), Bias::Right, &());
        assert_eq!(cursor.item(), Some(&2));

        // Slicing without resetting starts from where the cursor is parked at.
        cursor.seek(&Count(1), Bias::Right, &());

        // 这里可以画图理解，slice 到的 item 是 exclusive 的
        assert_eq!(
            cursor.slice(&Count(3), Bias::Right, &()).items(&()),
            vec![2, 3]
        );
        assert_eq!(
            cursor.slice(&Count(6), Bias::Left, &()).items(&()),
            vec![4, 5]
        );
        assert_eq!(
            cursor.slice(&Count(6), Bias::Right, &()).items(&()),
            vec![6]
        );
    }

    #[test]
    fn test_edit() {
        let mut tree = SumTree::<u8>::new();

        let removed = tree.edit(vec![Edit::Insert(1), Edit::Insert(2), Edit::Insert(0)], &());
        assert_eq!(tree.items(&()), vec![0, 1, 2]);
        assert_eq!(removed, Vec::<u8>::new());
        assert_eq!(tree.get(&0, &()), Some(&0));
        assert_eq!(tree.get(&1, &()), Some(&1));
        assert_eq!(tree.get(&2, &()), Some(&2));
        assert_eq!(tree.get(&4, &()), None);

        let removed = tree.edit(vec![Edit::Insert(2), Edit::Insert(4), Edit::Remove(0)], &());
        assert_eq!(tree.items(&()), vec![1, 2, 4]);
        assert_eq!(removed, vec![0, 2]);
        assert_eq!(tree.get(&0, &()), None);
        assert_eq!(tree.get(&1, &()), Some(&1));
        assert_eq!(tree.get(&2, &()), Some(&2));
        assert_eq!(tree.get(&4, &()), Some(&4));
    }

    #[test]
    fn test_from_iter() {
        assert_eq!(
            SumTree::from_iter(0..100, &()).items(&()),
            (0..100).collect::<Vec<_>>()
        );

        // Ensure `from_iter` works correctly when the given iterator restarts
        // after calling `next` if `None` was already returned.
        let mut ix = 0;
        let iterator = std::iter::from_fn(|| {
            ix = (ix + 1) % 2;
            if ix == 1 {
                Some(1)
            } else {
                None
            }
        });
        assert_eq!(SumTree::from_iter(iterator, &()).items(&()), vec![1]);
    }

    #[derive(Clone, Default, Debug)]
    pub struct IntegersSummary {
        count: usize,
        sum: usize,
        contains_even: bool,
        max: u8,
    }

    #[derive(Ord, PartialOrd, Default, Eq, PartialEq, Clone, Debug)]
    struct Count(usize);

    #[derive(Ord, PartialOrd, Default, Eq, PartialEq, Clone, Debug)]
    struct Sum(usize);

    impl Item for u8 {
        type Summary = IntegersSummary;

        fn summary(&self) -> Self::Summary {
            IntegersSummary {
                count: 1,
                sum: *self as usize,
                contains_even: (*self & 1) == 0,
                max: *self,
            }
        }
    }

    impl KeyedItem for u8 {
        type Key = u8;

        fn key(&self) -> Self::Key {
            *self
        }
    }

    impl Summary for IntegersSummary {
        type Context = ();

        fn add_summary(&mut self, other: &Self, _: &()) {
            self.count += other.count;
            self.sum += other.sum;
            self.contains_even |= other.contains_even;
            self.max = cmp::max(self.max, other.max);
        }
    }

    impl<'a> Dimension<'a, IntegersSummary> for u8 {
        fn add_summary(&mut self, summary: &IntegersSummary, _: &()) {
            *self = summary.max;
        }
    }

    impl<'a> Dimension<'a, IntegersSummary> for Count {
        fn add_summary(&mut self, summary: &IntegersSummary, _: &()) {
            self.0 += summary.count;
        }
    }

    impl<'a> SeekTarget<'a, IntegersSummary, IntegersSummary> for Count {
        fn cmp(&self, cursor_location: &IntegersSummary, _: &()) -> Ordering {
            self.0.cmp(&cursor_location.count)
        }
    }

    impl<'a> Dimension<'a, IntegersSummary> for Sum {
        fn add_summary(&mut self, summary: &IntegersSummary, _: &()) {
            self.0 += summary.sum;
        }
    }
}
```

以下是 rope.rs 的部分代码分析：
```rust
// rope.rs
mod offset_utf16;
mod point;
mod point_utf16;
mod unclipped;

use arrayvec::ArrayString;
use smallvec::SmallVec;
use std::{
    cmp, fmt, io, mem,
    ops::{AddAssign, Range},
    str,
};
use sum_tree::{Bias, Dimension, SumTree};
use unicode_segmentation::GraphemeCursor;
use util::debug_panic;

pub use offset_utf16::OffsetUtf16;
pub use point::Point;
pub use point_utf16::PointUtf16;
pub use unclipped::Unclipped;

#[cfg(test)]
const CHUNK_BASE: usize = 6;

#[cfg(not(test))]
const CHUNK_BASE: usize = 64;

#[derive(Clone, Default)]
pub struct Rope {
    chunks: SumTree<Chunk>,
}

impl Rope {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn append(&mut self, rope: Rope) {
        let mut chunks = rope.chunks.cursor::<()>();
        chunks.next(&());
        if let Some(chunk) = chunks.item() {
            if self.chunks.last().map_or(false, |c| c.0.len() < CHUNK_BASE)
                || chunk.0.len() < CHUNK_BASE
            {
                self.push(&chunk.0);
                chunks.next(&());
            }
        }

        self.chunks.append(chunks.suffix(&()), &());
        self.check_invariants();
    }

    pub fn replace(&mut self, range: Range<usize>, text: &str) {
        let mut new_rope = Rope::new();
        let mut cursor = self.cursor(0);
        new_rope.append(cursor.slice(range.start));
        cursor.seek_forward(range.end);
        new_rope.push(text);
        new_rope.append(cursor.suffix());
        *self = new_rope;
    }

    pub fn slice(&self, range: Range<usize>) -> Rope {
        let mut cursor = self.cursor(0);
        cursor.seek_forward(range.start);
        cursor.slice(range.end)
    }

    pub fn slice_rows(&self, range: Range<u32>) -> Rope {
        // This would be more efficient with a forward advance after the first, but it's fine.
        let start = self.point_to_offset(Point::new(range.start, 0));
        let end = self.point_to_offset(Point::new(range.end, 0));
        self.slice(start..end)
    }

    pub fn push(&mut self, mut text: &str) {
        // if the two can be put into a single item, then do it
        // otherwise try make the first size CHUNK_BASE.
        self.chunks.update_last(
            |last_chunk| {
                let split_ix = if last_chunk.0.len() + text.len() <= 2 * CHUNK_BASE {
                    text.len()
                } else {
                    let mut split_ix =
                        cmp::min(CHUNK_BASE.saturating_sub(last_chunk.0.len()), text.len());
                    while !text.is_char_boundary(split_ix) {
                        split_ix += 1;
                    }
                    split_ix
                };

                let (suffix, remainder) = text.split_at(split_ix);
                last_chunk.0.push_str(suffix);
                text = remainder;
            },
            &(),
        );

        if text.len() > 2048 {
            return self.push_large(text);
        }
        let mut new_chunks = SmallVec::<[_; 16]>::new();

        while !text.is_empty() {
            let mut split_ix = cmp::min(2 * CHUNK_BASE, text.len());
            while !text.is_char_boundary(split_ix) {
                split_ix -= 1;
            }
            let (chunk, remainder) = text.split_at(split_ix);
            new_chunks.push(Chunk(ArrayString::from(chunk).unwrap()));
            text = remainder;
        }

        #[cfg(test)]
        const PARALLEL_THRESHOLD: usize = 4;
        #[cfg(not(test))]
        const PARALLEL_THRESHOLD: usize = 4 * (2 * sum_tree::TREE_BASE);

        if new_chunks.len() >= PARALLEL_THRESHOLD {
            self.chunks.par_extend(new_chunks.into_vec(), &());
        } else {
            self.chunks.extend(new_chunks, &());
        }

        self.check_invariants();
    }

    /// A copy of `push` specialized for working with large quantities of text.
    fn push_large(&mut self, mut text: &str) {
        // To avoid frequent reallocs when loading large swaths of file contents,
        // we estimate worst-case `new_chunks` capacity;
        // Chunk is a fixed-capacity buffer. If a character falls on
        // chunk boundary, we push it off to the following chunk (thus leaving a small bit of capacity unfilled in current chunk).
        // Worst-case chunk count when loading a file is then a case where every chunk ends up with that unused capacity.
        // Since we're working with UTF-8, each character is at most 4 bytes wide. It follows then that the worst case is where
        // a chunk ends with 3 bytes of a 4-byte character. These 3 bytes end up being stored in the following chunk, thus wasting
        // 3 bytes of storage in current chunk.
        // For example, a 1024-byte string can occupy between 32 (full ASCII, 1024/32) and 36 (full 4-byte UTF-8, 1024 / 29 rounded up) chunks.
        const MIN_CHUNK_SIZE: usize = 2 * CHUNK_BASE - 3;

        // We also round up the capacity up by one, for a good measure; we *really* don't want to realloc here, as we assume that the # of characters
        // we're working with there is large.
        let capacity = (text.len() + MIN_CHUNK_SIZE - 1) / MIN_CHUNK_SIZE;
        let mut new_chunks = Vec::with_capacity(capacity);

        while !text.is_empty() {
            let mut split_ix = cmp::min(2 * CHUNK_BASE, text.len());
            while !text.is_char_boundary(split_ix) {
                split_ix -= 1;
            }
            let (chunk, remainder) = text.split_at(split_ix);
            new_chunks.push(Chunk(ArrayString::from(chunk).unwrap()));
            text = remainder;
        }

        #[cfg(test)]
        const PARALLEL_THRESHOLD: usize = 4;
        #[cfg(not(test))]
        const PARALLEL_THRESHOLD: usize = 4 * (2 * sum_tree::TREE_BASE);

        if new_chunks.len() >= PARALLEL_THRESHOLD {
            self.chunks.par_extend(new_chunks, &());
        } else {
            self.chunks.extend(new_chunks, &());
        }

        self.check_invariants();
    }
    pub fn push_front(&mut self, text: &str) {
        let suffix = mem::replace(self, Rope::from(text));
        self.append(suffix);
    }

    fn check_invariants(&self) {
        #[cfg(test)]
        {
            // Ensure all chunks except maybe the last one are not underflowing.
            // Allow some wiggle room for multibyte characters at chunk boundaries.
            let mut chunks = self.chunks.cursor::<()>().peekable();
            while let Some(chunk) = chunks.next() {
                if chunks.peek().is_some() {
                    // utf-8 max size for single char is 4 bytes
                    assert!(chunk.0.len() + 3 >= CHUNK_BASE);
                }
            }
        }
    }

    pub fn summary(&self) -> TextSummary {
        self.chunks.summary().text.clone()
    }

    pub fn len(&self) -> usize {
        self.chunks.extent(&())
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }

    pub fn max_point(&self) -> Point {
        self.chunks.extent(&())
    }

    pub fn max_point_utf16(&self) -> PointUtf16 {
        self.chunks.extent(&())
    }

    pub fn cursor(&self, offset: usize) -> Cursor {
        Cursor::new(self, offset)
    }

    pub fn chars(&self) -> impl Iterator<Item = char> + '_ {
        self.chars_at(0)
    }

    pub fn chars_at(&self, start: usize) -> impl Iterator<Item = char> + '_ {
        self.chunks_in_range(start..self.len()).flat_map(str::chars)
    }

    pub fn reversed_chars_at(&self, start: usize) -> impl Iterator<Item = char> + '_ {
        self.reversed_chunks_in_range(0..start)
            .flat_map(|chunk| chunk.chars().rev())
    }

    pub fn bytes_in_range(&self, range: Range<usize>) -> Bytes {
        Bytes::new(self, range, false)
    }

    pub fn reversed_bytes_in_range(&self, range: Range<usize>) -> Bytes {
        Bytes::new(self, range, true)
    }

    pub fn chunks(&self) -> Chunks {
        self.chunks_in_range(0..self.len())
    }

    pub fn chunks_in_range(&self, range: Range<usize>) -> Chunks {
        Chunks::new(self, range, false)
    }

    pub fn reversed_chunks_in_range(&self, range: Range<usize>) -> Chunks {
        Chunks::new(self, range, true)
    }

    pub fn offset_to_offset_utf16(&self, offset: usize) -> OffsetUtf16 {
        if offset >= self.summary().len {
            return self.summary().len_utf16;
        }
        let mut cursor = self.chunks.cursor::<(usize, OffsetUtf16)>();
        cursor.seek(&offset, Bias::Left, &());
        let overshoot = offset - cursor.start().0;
        cursor.start().1
            + cursor.item().map_or(Default::default(), |chunk| {
                chunk.offset_to_offset_utf16(overshoot)
            })
    }

    pub fn offset_utf16_to_offset(&self, offset: OffsetUtf16) -> usize {
        if offset >= self.summary().len_utf16 {
            return self.summary().len;
        }
        let mut cursor = self.chunks.cursor::<(OffsetUtf16, usize)>();
        cursor.seek(&offset, Bias::Left, &());
        let overshoot = offset - cursor.start().0;
        cursor.start().1
            + cursor.item().map_or(Default::default(), |chunk| {
                chunk.offset_utf16_to_offset(overshoot)
            })
    }

    pub fn offset_to_point(&self, offset: usize) -> Point {
        if offset >= self.summary().len {
            return self.summary().lines;
        }
        let mut cursor = self.chunks.cursor::<(usize, Point)>();
        cursor.seek(&offset, Bias::Left, &());
        let overshoot = offset - cursor.start().0;
        cursor.start().1
            + cursor
                .item()
                .map_or(Point::zero(), |chunk| chunk.offset_to_point(overshoot))
    }

    pub fn offset_to_point_utf16(&self, offset: usize) -> PointUtf16 {
        if offset >= self.summary().len {
            return self.summary().lines_utf16();
        }
        let mut cursor = self.chunks.cursor::<(usize, PointUtf16)>();
        cursor.seek(&offset, Bias::Left, &());
        let overshoot = offset - cursor.start().0;
        cursor.start().1
            + cursor.item().map_or(PointUtf16::zero(), |chunk| {
                chunk.offset_to_point_utf16(overshoot)
            })
    }

    pub fn point_to_point_utf16(&self, point: Point) -> PointUtf16 {
        if point >= self.summary().lines {
            return self.summary().lines_utf16();
        }
        let mut cursor = self.chunks.cursor::<(Point, PointUtf16)>();
        cursor.seek(&point, Bias::Left, &());
        let overshoot = point - cursor.start().0;
        cursor.start().1
            + cursor.item().map_or(PointUtf16::zero(), |chunk| {
                chunk.point_to_point_utf16(overshoot)
            })
    }

    pub fn point_to_offset(&self, point: Point) -> usize {
        if point >= self.summary().lines {
            return self.summary().len;
        }
        let mut cursor = self.chunks.cursor::<(Point, usize)>();
        cursor.seek(&point, Bias::Left, &());
        let overshoot = point - cursor.start().0;
        cursor.start().1
            + cursor
                .item()
                .map_or(0, |chunk| chunk.point_to_offset(overshoot))
    }

    pub fn point_utf16_to_offset(&self, point: PointUtf16) -> usize {
        self.point_utf16_to_offset_impl(point, false)
    }

    pub fn unclipped_point_utf16_to_offset(&self, point: Unclipped<PointUtf16>) -> usize {
        self.point_utf16_to_offset_impl(point.0, true)
    }

    fn point_utf16_to_offset_impl(&self, point: PointUtf16, clip: bool) -> usize {
        if point >= self.summary().lines_utf16() {
            return self.summary().len;
        }
        let mut cursor = self.chunks.cursor::<(PointUtf16, usize)>();
        cursor.seek(&point, Bias::Left, &());
        let overshoot = point - cursor.start().0;
        cursor.start().1
            + cursor
                .item()
                .map_or(0, |chunk| chunk.point_utf16_to_offset(overshoot, clip))
    }

    pub fn unclipped_point_utf16_to_point(&self, point: Unclipped<PointUtf16>) -> Point {
        if point.0 >= self.summary().lines_utf16() {
            return self.summary().lines;
        }
        let mut cursor = self.chunks.cursor::<(PointUtf16, Point)>();
        cursor.seek(&point.0, Bias::Left, &());
        let overshoot = Unclipped(point.0 - cursor.start().0);
        cursor.start().1
            + cursor.item().map_or(Point::zero(), |chunk| {
                chunk.unclipped_point_utf16_to_point(overshoot)
            })
    }

    pub fn clip_offset(&self, mut offset: usize, bias: Bias) -> usize {
        let mut cursor = self.chunks.cursor::<usize>();
        cursor.seek(&offset, Bias::Left, &());
        if let Some(chunk) = cursor.item() {
            let mut ix = offset - cursor.start();
            while !chunk.0.is_char_boundary(ix) {
                match bias {
                    Bias::Left => {
                        ix -= 1;
                        offset -= 1;
                    }
                    Bias::Right => {
                        ix += 1;
                        offset += 1;
                    }
                }
            }
            offset
        } else {
            self.summary().len
        }
    }

    pub fn clip_offset_utf16(&self, offset: OffsetUtf16, bias: Bias) -> OffsetUtf16 {
        let mut cursor = self.chunks.cursor::<OffsetUtf16>();
        cursor.seek(&offset, Bias::Right, &());
        if let Some(chunk) = cursor.item() {
            let overshoot = offset - cursor.start();
            *cursor.start() + chunk.clip_offset_utf16(overshoot, bias)
        } else {
            self.summary().len_utf16
        }
    }

    pub fn clip_point(&self, point: Point, bias: Bias) -> Point {
        let mut cursor = self.chunks.cursor::<Point>();
        cursor.seek(&point, Bias::Right, &());
        if let Some(chunk) = cursor.item() {
            let overshoot = point - cursor.start();
            *cursor.start() + chunk.clip_point(overshoot, bias)
        } else {
            self.summary().lines
        }
    }

    pub fn clip_point_utf16(&self, point: Unclipped<PointUtf16>, bias: Bias) -> PointUtf16 {
        let mut cursor = self.chunks.cursor::<PointUtf16>();
        cursor.seek(&point.0, Bias::Right, &());
        if let Some(chunk) = cursor.item() {
            let overshoot = Unclipped(point.0 - cursor.start());
            *cursor.start() + chunk.clip_point_utf16(overshoot, bias)
        } else {
            self.summary().lines_utf16()
        }
    }

    pub fn line_len(&self, row: u32) -> u32 {
        self.clip_point(Point::new(row, u32::MAX), Bias::Left)
            .column
    }
}

impl<'a> From<&'a str> for Rope {
    fn from(text: &'a str) -> Self {
        let mut rope = Self::new();
        rope.push(text);
        rope
    }
}

impl<'a> FromIterator<&'a str> for Rope {
    fn from_iter<T: IntoIterator<Item = &'a str>>(iter: T) -> Self {
        let mut rope = Rope::new();
        for chunk in iter {
            rope.push(chunk);
        }
        rope
    }
}

impl From<String> for Rope {
    fn from(text: String) -> Self {
        Rope::from(text.as_str())
    }
}

impl fmt::Display for Rope {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        for chunk in self.chunks() {
            write!(f, "{}", chunk)?;
        }
        Ok(())
    }
}

impl fmt::Debug for Rope {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        use std::fmt::Write as _;

        write!(f, "\"")?;
        let mut format_string = String::new();
        for chunk in self.chunks() {
            write!(&mut format_string, "{:?}", chunk)?;
            write!(f, "{}", &format_string[1..format_string.len() - 1])?;
            format_string.clear();
        }
        write!(f, "\"")?;
        Ok(())
    }
}

pub struct Cursor<'a> {
    rope: &'a Rope,
    chunks: sum_tree::Cursor<'a, Chunk, usize>,
    // position Dimension
    offset: usize,
}

impl<'a> Cursor<'a> {
    pub fn new(rope: &'a Rope, offset: usize) -> Self {
        let mut chunks = rope.chunks.cursor();
        chunks.seek(&offset, Bias::Right, &());
        Self {
            rope,
            chunks,
            offset,
        }
    }

    pub fn seek_forward(&mut self, end_offset: usize) {
        debug_assert!(end_offset >= self.offset);

        self.chunks.seek_forward(&end_offset, Bias::Right, &());
        self.offset = end_offset;
    }

    pub fn slice(&mut self, end_offset: usize) -> Rope {
        debug_assert!(
            end_offset >= self.offset,
            "cannot slice backwards from {} to {}",
            self.offset,
            end_offset
        );

        let mut slice = Rope::new();
        // 当前item内部
        if let Some(start_chunk) = self.chunks.item() {
            let start_ix = self.offset - self.chunks.start();
            let end_ix = cmp::min(end_offset, self.chunks.end(&())) - self.chunks.start();
            slice.push(&start_chunk.0[start_ix..end_ix]);
        }

        // 超过当前item的范围
        if end_offset > self.chunks.end(&()) {
            self.chunks.next(&());
            // 完整的items被加进来(SumTree Cursor's slice function只会加完整的items)
            slice.append(Rope {
                chunks: self.chunks.slice(&end_offset, Bias::Right, &()),
            });
            // 最后不完整的那个加进来
            if let Some(end_chunk) = self.chunks.item() {
                let end_ix = end_offset - self.chunks.start();
                slice.push(&end_chunk.0[..end_ix]);
            }
        }

        self.offset = end_offset;
        slice
    }

    pub fn summary<D: TextDimension>(&mut self, end_offset: usize) -> D {
        debug_assert!(end_offset >= self.offset);

        let mut summary = D::default();
        // curr item
        if let Some(start_chunk) = self.chunks.item() {
            let start_ix = self.offset - self.chunks.start();
            let end_ix = cmp::min(end_offset, self.chunks.end(&())) - self.chunks.start();
            summary.add_assign(&D::from_text_summary(&TextSummary::from(
                &start_chunk.0[start_ix..end_ix],
            )));
        }

        if end_offset > self.chunks.end(&()) {
            self.chunks.next(&());
            // add whole item Summary
            summary.add_assign(&self.chunks.summary(&end_offset, Bias::Right, &()));
            // 最后不完整的那个加进来
            if let Some(end_chunk) = self.chunks.item() {
                let end_ix = end_offset - self.chunks.start();
                summary.add_assign(&D::from_text_summary(&TextSummary::from(
                    &end_chunk.0[..end_ix],
                )));
            }
        }

        self.offset = end_offset;
        summary
    }

    pub fn suffix(mut self) -> Rope {
        self.slice(self.rope.chunks.extent(&()))
    }

    pub fn offset(&self) -> usize {
        self.offset
    }
}

pub struct Chunks<'a> {
    chunks: sum_tree::Cursor<'a, Chunk, usize>,
    range: Range<usize>,
    // absolut offset within whole Rope
    offset: usize,
    // determines the direction of iteration
    reversed: bool,
}

impl<'a> Chunks<'a> {
    pub fn new(rope: &'a Rope, range: Range<usize>, reversed: bool) -> Self {
        let mut chunks = rope.chunks.cursor();
        let offset = if reversed {
            chunks.seek(&range.end, Bias::Left, &());
            range.end
        } else {
            // use Bias::Right because when target pos is is item boundary, go to fresh started item
            chunks.seek(&range.start, Bias::Right, &());
            range.start
        };
        Self {
            chunks,
            range,
            offset,
            reversed,
        }
    }

    fn offset_is_valid(&self) -> bool {
        if self.reversed {
            if self.offset <= self.range.start || self.offset > self.range.end {
                return false;
            }
        } else {
            if self.offset < self.range.start || self.offset >= self.range.end {
                return false;
            }
        }

        true
    }

    pub fn offset(&self) -> usize {
        self.offset
    }

    pub fn seek(&mut self, mut offset: usize) {
        offset = offset.clamp(self.range.start, self.range.end);

        let bias = if self.reversed {
            Bias::Left
        } else {
            Bias::Right
        };

        if offset >= self.chunks.end(&()) {
            self.chunks.seek_forward(&offset, bias, &());
        } else {
            self.chunks.seek(&offset, bias, &());
        }

        self.offset = offset;
    }

    pub fn set_range(&mut self, range: Range<usize>) {
        self.range = range.clone();
        self.seek(range.start);
    }

    /// Moves this cursor to the start of the next line in the rope.
    ///
    /// This method advances the cursor to the beginning of the next line.
    /// If the cursor is already at the end of the rope, this method does nothing.
    /// Reversed chunks iterators are not currently supported and will panic.
    ///
    /// Returns `true` if the cursor was successfully moved to the next line start,
    /// or `false` if the cursor was already at the end of the rope.
    pub fn next_line(&mut self) -> bool {
        assert!(!self.reversed);

        let mut found = false;
        if let Some(chunk) = self.peek() {
            if let Some(newline_ix) = chunk.find('\n') {
                self.offset += newline_ix + 1;
                found = self.offset <= self.range.end;
            } else {
                self.chunks
                    .search_forward(|summary| summary.text.lines.row > 0, &());
                self.offset = *self.chunks.start();

                if let Some(newline_ix) = self.peek().and_then(|chunk| chunk.find('\n')) {
                    self.offset += newline_ix + 1;
                    found = self.offset <= self.range.end;
                } else {
                    // no more newlines. Move to the very end
                    self.offset = self.chunks.end(&());
                }
            }
            // this line can be moved into prev `if let`
            if self.offset == self.chunks.end(&()) {
                self.next();
            }
        }

        if self.offset > self.range.end {
            self.offset = cmp::min(self.offset, self.range.end);
            self.chunks.seek(&self.offset, Bias::Right, &());
        }

        found
    }

    /// Move this cursor to the preceding position in the rope that starts a new line.
    /// Reversed chunks iterators are not currently supported and will panic.
    ///
    /// If this cursor is not on the start of a line, it will be moved to the start of
    /// its current line. Otherwise it will be moved to the start of the previous line.
    /// It updates the cursor's position and returns true if a previous line was found,
    /// or false if the cursor was already at the start of the rope.
    pub fn prev_line(&mut self) -> bool {
        assert!(!self.reversed);

        let initial_offset = self.offset;

        if self.offset == *self.chunks.start() {
            self.chunks.prev(&());
        }

        if let Some(chunk) = self.chunks.item() {
            let mut end_ix = self.offset - *self.chunks.start();
            if chunk.0.as_bytes()[end_ix - 1] == b'\n' {
                end_ix -= 1;
            }

            if let Some(newline_ix) = chunk.0[..end_ix].rfind('\n') {
                self.offset = *self.chunks.start() + newline_ix + 1;
                if self.offset_is_valid() {
                    return true;
                }
            }
        }

        self.chunks
            .search_backward(|summary| summary.text.lines.row > 0, &());
        self.offset = *self.chunks.start();
        if let Some(chunk) = self.chunks.item() {
            if let Some(newline_ix) = chunk.0.rfind('\n') {
                self.offset += newline_ix + 1;
                if self.offset_is_valid() {
                    if self.offset == self.chunks.end(&()) {
                        self.chunks.next(&());
                    }

                    return true;
                }
            }
        }

        if !self.offset_is_valid() || self.chunks.item().is_none() {
            self.offset = self.range.start;
            self.chunks.seek(&self.offset, Bias::Right, &());
        }

        self.offset < initial_offset && self.offset == 0
    }

    // curr item union range
    pub fn peek(&self) -> Option<&'a str> {
        if !self.offset_is_valid() {
            return None;
        }

        let chunk = self.chunks.item()?;
        let chunk_start = *self.chunks.start();
        let slice_range = if self.reversed {
            let slice_start = cmp::max(chunk_start, self.range.start) - chunk_start;
            let slice_end = self.offset - chunk_start;
            slice_start..slice_end
        } else {
            let slice_start = self.offset - chunk_start;
            let slice_end = cmp::min(self.chunks.end(&()), self.range.end) - chunk_start;
            slice_start..slice_end
        };

        Some(&chunk.0[slice_range])
    }

    pub fn lines(self) -> Lines<'a> {
        let reversed = self.reversed;
        Lines {
            chunks: self,
            current_line: String::new(),
            done: false,
            reversed,
        }
    }
}

impl<'a> Iterator for Chunks<'a> {
    type Item = &'a str;

    fn next(&mut self) -> Option<Self::Item> {
        let chunk = self.peek()?;
        if self.reversed {
            self.offset -= chunk.len();
            if self.offset <= *self.chunks.start() {
                self.chunks.prev(&());
            }
        } else {
            self.offset += chunk.len();
            if self.offset >= self.chunks.end(&()) {
                self.chunks.next(&());
            }
        }

        Some(chunk)
    }
}

pub struct Bytes<'a> {
    chunks: sum_tree::Cursor<'a, Chunk, usize>,
    range: Range<usize>,
    reversed: bool,
}

impl<'a> Bytes<'a> {
    pub fn new(rope: &'a Rope, range: Range<usize>, reversed: bool) -> Self {
        let mut chunks = rope.chunks.cursor();
        if reversed {
            chunks.seek(&range.end, Bias::Left, &());
        } else {
            chunks.seek(&range.start, Bias::Right, &());
        }
        Self {
            chunks,
            range,
            reversed,
        }
    }

    // curr item union range
    pub fn peek(&self) -> Option<&'a [u8]> {
        let chunk = self.chunks.item()?;
        if self.reversed && self.range.start >= self.chunks.end(&()) {
            return None;
        }
        let chunk_start = *self.chunks.start();
        if self.range.end <= chunk_start {
            return None;
        }
        let start = self.range.start.saturating_sub(chunk_start);
        let end = self.range.end - chunk_start;
        Some(&chunk.0.as_bytes()[start..chunk.0.len().min(end)])
    }
}

impl<'a> Iterator for Bytes<'a> {
    type Item = &'a [u8];

    fn next(&mut self) -> Option<Self::Item> {
        let result = self.peek();
        if result.is_some() {
            if self.reversed {
                self.chunks.prev(&());
            } else {
                self.chunks.next(&());
            }
        }
        result
    }
}

// read curr item into buf
impl<'a> io::Read for Bytes<'a> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        if let Some(chunk) = self.peek() {
            let len = cmp::min(buf.len(), chunk.len());
            if self.reversed {
                buf[..len].copy_from_slice(&chunk[chunk.len() - len..]);
                buf[..len].reverse();
                self.range.end -= len;
            } else {
                buf[..len].copy_from_slice(&chunk[..len]);
                self.range.start += len;
            }

            if len == chunk.len() {
                if self.reversed {
                    self.chunks.prev(&());
                } else {
                    self.chunks.next(&());
                }
            }
            Ok(len)
        } else {
            Ok(0)
        }
    }
}
```
## 总结

Zed 编辑器通过使用 Rope 和底层的 SumTree 数据结构，实现了高效的文本表示和操作。
这种设计不仅允许快速的插入、删除和修改操作，还支持复杂的统计和查找功能。
通过 Summary、Dimension 和 SeekTarget 等概念的巧妙运用，Zed 能够灵活地处理各种文本操作需求，
同时保持良好的性能。
