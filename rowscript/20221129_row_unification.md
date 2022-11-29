# Row Polymorphism 和 Unification

* Anqur
* 2022-11-29

## 重新思考 Unification 的算法

在 [TinyIdris] 的讲解中, Edwin Brady 提到了 unification 算法的几个步骤, 以及相关的结构:

* 一个 definition 可以是空的 [`None`], 表示它是一个 declaration (signature), 但是还没被
  define
* 在 elaborate 表达式的过程中, elaborator 会生成 meta variables, [把 meta 塞到全局],
  其中 `Hole` 是一种 definition
* TODO

在 [前文] 中, 我们有个这样的例子 (删掉了一些比较奇怪的符号):

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
    → {auto q : (Field "n" number) ≤ $C}
    → (x : Object a) → (y : Object b)
    → number
getNum = {??}
```

TODO

[TinyIdris]: https://www.youtube.com/watch?v=9SKN_vTQ1xM
[`None`]: https://github.com/edwinb/SPLV20/blob/c6db8f38ee6e54fffd15cbca1ad3fe64060775b0/TinyIdris-v2/src/Core/Context.idr#L12
[把 meta 塞到全局]: https://github.com/edwinb/SPLV20/blob/b401de02b483d9e43f482f6f9d61c431332d1b75/TinyIdris-v2/src/TTImp/Elab/Term.idr#L133
[前文]: ./20221128_row_theory_prereq.md
