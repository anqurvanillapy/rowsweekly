# Row Polymorphism 和 Unification

* Anqur
* 2022-11-29

## 重新思考 Unification 的算法

在 [TinyIdris] 的讲解中, Edwin Brady 提到了 unification 算法的几个步骤, 以及相关的结构:

* 一个 definition 可以是空的 [`None`], 表示它是一个 declaration (signature), 但是还没被
  define, 感觉可以用于 recursive definition
* 在 elaborate definition 的过程中, 作为 [`Guess`] 插入到全局 definition 表中
* 在 elaborate 具体的表达式时, 展开 implicits, [生成 meta variables 塞到全局], 其中
  `Hole` 是一种 definition
* 对每一个表达式针对性地 solve constraints

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

函数的实际参数 (`{n: 42}`) 应该是可以直接 infer 出来的. 接着, 形式参数的类型 (guess),
和实际参数的类型 (constraint), 便可以进行 unify:

```agda
Object ?a = Object (Field "n" number)
```

解出:

* `?a : Row = Field "n" number` (直接可得)
* `?p : c ≡ (Field "n" number) ⨀ b` (进行 substitution, 然后继续 stuck)

下一个检查的式子 (为了防止歧义, 这里特别标注了是第二个 function application):

```agda
        getNum {n: 42} {m: 69}
--      ^------------^ ^-----^
```

同理得到:

```agda
Object ?b = Object (Field "m" number)
```

解出:

* `?b : Row = Field "m" number` (直接解得)
* `?c : Row = More "n" number (Field "m" number)` (等量代换)
* `?p : (More "n" number (Field "m" number)) ≡ (Field "n" number) ⨀ (Field "m" number) = RowRefl`
  (substitution, 并根据 auto implicit 从 data constructor 中 freely generate 出来)
* `?q : (Field "n" number) ≤ (More "n" number (Field "m" number)) = RowSat`
  (substitution, 并从 data constructor 中 freely generate 出来)

至此 unification state 应该为结束, 因为没有 guesses 和 constraints 了.

[TinyIdris]: https://www.youtube.com/watch?v=9SKN_vTQ1xM
[`None`]: https://github.com/edwinb/SPLV20/blob/b401de02b483d9e43f482f6f9d61c431332d1b75/TinyIdris-v2/src/Core/Context.idr#L12
[`Guess`]: https://github.com/edwinb/SPLV20/blob/b401de02b483d9e43f482f6f9d61c431332d1b75/TinyIdris-v2/src/Core/Context.idr#L18
[生成 meta variables 塞到全局]: https://github.com/edwinb/SPLV20/blob/b401de02b483d9e43f482f6f9d61c431332d1b75/TinyIdris-v2/src/TTImp/Elab/Term.idr#L133
[前文]: ./20221128_row_theory_prereq.md
