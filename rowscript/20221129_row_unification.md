# Row Polymorphism 和 Unification

* Anqur
* 2022-11-29

## 重新思考 Unification 的算法

在 [TinyIdris] 的讲解中, Edwin Brady 提到了 unification 算法的几个步骤, 以及相关的结构:

* 一个 definition 可以是空的 [`None`], 表示它是一个 declaration (signature), 但是还没被
  define, 感觉可以用于 recursive definition
* TODO
* 在 elaborate 表达式的过程中, elaborator 会生成 meta variables, [把 meta 塞到全局],
  其中 `Hole` 是一种 definition
* TODO

在 [前文] 中, 我们有个这样的例子 (删掉了一些比较奇怪的符号), 用比较完整的 Agda 代码表示:

```agda
data Row : Set where
    Field : (n : Name) → (ty : Set) → Row
    More : (n : Name) → (ty : Set) → Row → Row

data _≡_⨀_ (c a b : Row) : Set where
    RowRefl : c ≡ a ⨀ b

data _≤_ (a b : Row) : Set where
    RowSat : a ≤ b

data Object (r : Row) : Set where
    MkObject : Object r

getNum
    : {a : Row} → {b : Row} → {c : Row}
    → {auto p : c ≡ a ⨀ b}
    → {auto q : (Field "n" number) ≤ c}
    → (x : Object a) → (y : Object b)
    → number
getNum = {??}
```

我们需要检查的 definition:

```agda
lol : number
lol = getNum {n: 42} {m: 69}
```

这个时候 `lol` 是一种叫做 [`Guess`] 的 definition, 它的 guess 就是下标注明的表达式, 即:

```agda
        getNum {n: 42} {m: 69}
--      ^--------------------^
```

它的 constraints 就是 `number` 这个 *表达式*. 接下来深入到表达式内部第一个被检查的子表达式,
是 `getNum` 这个 function reference.

```agda
        getNum {n: 42} {m: 69}
--      ^----^
```

用 `<>` 符号表示 implicit arguments, 此时应该对表达式进行补全:

```plaintext
getNum <a = ?a, b = ?b, c = ?c, p = ?p, q = ?q> {n: 42} {m: 69}
```

这时, elaborator 应该生成如下几个 hole 塞入到全局:

* `?a : Row`
* `?b : Row`
* `?c : Row`
* `?p : c ≡ a ⨀ b`
* `?q : (Field "n" number) ≤ c`

下一个要检查的表达式是 function application.

```agda
        getNum {n: 42} {m: 69}
--      ^------------^

```

TODO

[TinyIdris]: https://www.youtube.com/watch?v=9SKN_vTQ1xM
[`None`]: https://github.com/edwinb/SPLV20/blob/b401de02b483d9e43f482f6f9d61c431332d1b75/TinyIdris-v2/src/Core/Context.idr#L12
[把 meta 塞到全局]: https://github.com/edwinb/SPLV20/blob/b401de02b483d9e43f482f6f9d61c431332d1b75/TinyIdris-v2/src/TTImp/Elab/Term.idr#L133
[前文]: ./20221128_row_theory_prereq.md
[`Guess`]: https://github.com/edwinb/SPLV20/blob/b401de02b483d9e43f482f6f9d61c431332d1b75/TinyIdris-v2/src/Core/Context.idr#L18
