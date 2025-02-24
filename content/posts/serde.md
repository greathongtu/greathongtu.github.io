+++
title = 'Serde'
date = 2025-02-24T16:53:16+08:00
draft = false
+++

## 序列化过程
首先通过 macro 为 struct 实现 impl Serialize。 在 impl 的 serialize 方法里，通过了解的结构信息（struct 有哪些 field，field 具体名称和类型等信息）调用对应的 Serializer 的方法。
也就是说 Serialize 负责控制如何将 Rust 数据结构转变为 serde data model（并不是真正的转变，只是控制如何转变：也就是如何调用 Serializer）。Serializer 负责具体执行（如serialize_i32, serialize_struct 方法等），将 serde data model 转变为具体的目标 format。
其中 Serializer 的实现一般在另一个库如 serde_json 里。 这也是为什么说 serde 库并不是 parse 库。

```rust
impl _serde::Serialize for Test {
    fn serialize<__S>(
        &self,
        __serializer: __S,
    ) -> _serde::__private::Result<__S::Ok, __S::Error>
    where
        __S: _serde::Serializer,
    {
        let mut __serde_state = _serde::Serializer::serialize_struct(
            __serializer,
            "Test",
            false as usize + 1 + 1,
        )?;
        _serde::ser::SerializeStruct::serialize_field(
            &mut __serde_state,
            "test_num",
            &self.test_num,
        )?;
        _serde::ser::SerializeStruct::serialize_field(
            &mut __serde_state,
            "test_str",
            &self.test_str,
        )?;
        _serde::ser::SerializeStruct::end(__serde_state)
    }
}
```

## 反序列化过程
首先通过 macro 为 struct 实现 impl Deserialize。 在 impl 的 deserialize 方法内，通过了解的信息调用相应的 deserializer_xxx 方法，对于 json 这样的自描述格式，直接调用 deserializer_any 方法就可以。同时还传递了一个 Visitor 进去，用来进行真正的反序列化工作。Visitor 的 associated type 就是最终反序列化的目标 Rust 数据结构格式，实现了 visit_seq、 visit_map 等方法用来给 Deserializer 调用。

Visitor 是 Deserialize 这一侧实现的，用来将 serde data model 转变为 Rust 数据结构。Vistor 的调用者 deserializer.deserializer_xxx 是 Deserializer 这一侧实现的，一般在另一个库如 serde_json 里，用来将不同的具体 format 转变为 serde data model。

还有辅助 FieldVisitor 的实现，用来帮助获取 map 的 key 信息。

```rust
// Deserialize::deserialize 方法
const FIELDS: &'static [&'static str] = &[
    "test_num",
    "test_str",
    "test_vector",
];
_serde::Deserializer::deserialize_struct(
    __deserializer,
    "Test",
    FIELDS,
    __Visitor {
        marker: _serde::__private::PhantomData::<Test>,
        lifetime: _serde::__private::PhantomData,
    },
)
```

```rust
// serde_json 的 deserialize_map 方法
fn deserialize_map<V>(self, visitor: V) -> Result<V::Value>
where
    V: de::Visitor<'de>,
{
    let peek = match tri!(self.parse_whitespace()) {
        Some(b) => b,
        None => {
            return Err(self.peek_error(ErrorCode::EofWhileParsingValue));
        }
    };

    let value = match peek {
        b'{' => {
            check_recursion! {
                self.eat_char();
                let ret = visitor.visit_map(MapAccess::new(self));
            }

            match (ret, self.end_map()) {
                (Ok(ret), Ok(())) => Ok(ret),
                (Err(err), _) | (_, Err(err)) => Err(err),
            }
        }
        _ => Err(self.peek_invalid_type(&visitor)),
    };

    match value {
        Ok(value) => Ok(value),
        Err(err) => Err(self.fix_position(err)),
    }
}
```

