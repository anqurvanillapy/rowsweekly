# 为什么 row theory 能行得通?

* Anqur
* 2022-12-20

我们将在这个文章阐述, 整个 row theory 的设计和 dependent types 融合在一起, 为什么是可行的?
具体的 idea 是什么? 有没有 PoC 代码证明 idea 是可以运行的呢?

## Row theory 的 idea

我们核心的 idea 在于:

* 使用基础的 MLTT 去表达 row 和 row predicates
* 借助一些高级的 elaboration 技术消除一些代码上的冗余:
  * Holes
  * Pattern unification
  * Implicit arguments
  * Auto implicit arguments

### 怎么用 MLTT 表达 row?

首先, 我们要知道什么是 row.

简单理解它就是 *成员* 这种东西, 或者说 *标签*, 它不仅能 *绑定* 数值 (比如 `(n: 42)`),
还能绑定类型 (比如 `(n: number)`), 和 object 组合的时候, 就成了 object type (比如
`{n : number}`), 也能和 enumeration 组合成 enum type (比如 `none | some(T)`, enum
type 具体语法我还没确定下来).

但需要注意的是, 我们在语法上, 数值表达式中 row 只能绑定数值, 类型表达式中 row 只能绑定类型,
完全是因为图方便和简单, 而且我们没必要让用户在语法上大量使用 dependent types.

深入理解时, 我们知道, 一个类型一般有 4 种规则, 我们以 unit 类型为例:

* **Type former**: 类型是怎么来的, 比如 `unit : type`, 我们说 `unit` 是一种类型
* **Term introduction rule**: 类型的值是怎么来的, 比如 `() : unit`, 当我们写 `()`
  这个表达式的时候, 这个表达式的类型是 `unit`
* **Term elimination rule**: 类型的值是怎么消去的, 对于 unit 类型来说, 一种消去的方法类似
  C++ 的 `(void)foo();`, 即 `let _ = tm in body`, 表达式 `tm` 的类型为 `unit`,
  然后它被 `_` 绑定 (被忽略了), 这个值不会被用到 `body` 里面去
* **Computation rule** (也叫 eta rule): 不要被 *computation* 这个词蒙骗,
  这个规则一般说的是数值和数值之间有没有什么相等的关系, 比如对于 lambda 我们知道表达式 `x` 和
  `(lambda y: y) x` 是一样的, 这个就是 lambda 的 computation rule; 对于 unit,
  我们一般说所有的 `()` 都是定义上相等的

那么 row 当然也就有这几个规则:

* Type former: 即 `row : type`, `row` 是个类型
* Term introduction rule: 即类似 `(n: number)` 或者 `(n: 42)` 这样的东西, 也可以很多个
  `(n: 42, m: "foo")`, 它们的类型都是 `row`, 即 `(n: number) : row`
* Term elimination rule:
  * Object type former, 即 `{_} : row -> type`, 塞入 `{}` 的中间就会变成 object type
    了, 例如 `{n : number} : type`
  * Enum type former, 即 `|_| : row -> type`, 塞入 `|_|` 变成 enum type, 例如
    `id(number) | name(string)`
* Computation rule: 比较 *成员名* 和 *值/类型* 是否相等就可以了, 就像比较两个哈希表的内容

### 那 row predicate 呢?

Row predicate 说的是两种和证明相关的关系:

* Row equality, 两个 row 是不是相等, 比如 `(n: number) = (m: string)` 这个就不成立,
  `(n: number, m: string) = (m: string, n: number)` 这个成立; 论文里面这个东西叫 row
  combination, 这是因为它是 `r1 = r2 + r3` 这种形式的, 我觉得不如干脆直接抽出那个等号来用
* Row ordering (论文里面叫 row containment), 判断 row 的包含关系, 比如
  `(n: number) <= (m: string, n: number)` 是成立的, 但是
  `(n: number, m: string) <= (n: string, m: string, o: bool)` 是不成立的, 因为 `n`
  的类型不对

那么我们就能知道:

* Row equality 本质上也是 row 的 elimination rule, 即 `_=_ : row -> row -> type`
* Row ordering 同上, 即 `_<=_ : row -> row -> type`, 我们还要有反着的 ordering,
  用户可能要用, 即 `_>=_ : row -> row -> type`

这两个类型有对应的 term introduction rule:

* Row reflexive, 表示相等关系满足, 即 `RowRefl : {A B : row} A = B`
* Row satisfied, 表示包含关系满足, 即 `RowSat : {A B : row} A <= B`, 或者
  `RowSat : {A B : row} A >= B`

那么, row 变量的输入, 以及 row predicate 的满足, 是怎么保证的呢?

### 简单的例子

我们用一个简单的程序来看 row 和 row predicate 是怎么使用起来的, 这里用 Agda 的语法.

假设我们有个函数 `f`, 它返回一个 `number` 类型的常量 `42`, 但是它要求输入一个带有成员
`(n: number)` 的 object:

```agda
f : (A : row) -> (p : (n: number) <: A) -> (o : object A) -> number
f _ _ _ = 42

example : number
example = f (n: number) RowRefl {n: 69}
-- 42
```

这里我们知道:

* `A` 就是 row 变量
* `p` 是 row predicate 的 proof, 它需要传入一个 `RowRefl`
* `o` 是 `object A` 这个 object type 下的一个值

这个函数的形式参数调换顺序, 也是可以 work 的:

```agda
f : (A : row) -> (o : object A) -> (p : (n : number) <: A) -> number
f _ _ _ = 42

example : number
example = f (n: number) {n: 69} RowRefl
```

那么问题就来了, 我们不可能让用户每次调用一个函数都传入 row 变量和 `RowRefl` 这个数值,
这个给用户的日常使用带来了相当大的噪音.

我们的 elaborator 怎么自动帮用户填上 row 和 row predicate proof 呢?

### 消除用户使用时的噪音

对于 row 的输入, 我们其实可以使用 implicit arguments 的特性, 将 `f` 改写为:

```agda
f : {A : row} -> {p : (n: number) <: A} -> (o : object A) -> number
f _ _ _ = 42

example : number
example = f {p = RowRefl} {n: 69}
```

可以发现, `A` 和 `p` 都被声明为 implicit parameter 了, 但是, 在使用的过程中, `example`
我们还是必须 `{p = RowRefl}` 显式地输入了 proof term, 这是因为 unification
的算法是不会主动为用户生成 term 的, 它只会创建一个 `(n: number) <: A` 类型的 constraint
(constraint 即 expected type, 这个表达式期盼的类型), 而用户要主动填入 guess (即符合
constraint 的一个值), 不主动填, 就会 stuck, 就会报 *unsolved meta* 的错误.

在 Agda 中, 解决一个 hole/goal 必须要主动去填 guess, 但是 Agsy 工具可以自动搜索出来. 在
Idris 2 中, 给 implicit parameter 标记 `auto`, 也可以自动搜索出答案,
它的搜索算法和优先级可参考 [Idris 2 Miscellany].

那么, 我们可以非常作弊地实现这个功能, 只为两种 row predicate 提供自动生成 term 的功能, 即:

```agda
f : {A : row} -> {auto p : (n: number) <: A} -> (o : object A) -> number
f _ _ _ = 42

example : number
example = f {n: 69}
```

**注意!!!** 这个代码是 work 的, 但是不是正确的!!!

这是因为, 我们的 `f` 是个柯里化的函数, 可以部份 apply, 当我们只输入 `A` 和 `p` 的两个参数后,
`p` 需要被立马创建出来, 这个时候 `auto` 就会直接给出 `A` 的答案, 而不是到输入 `o` 以后才被
`object A` 这个表达式反推出答案.

也就是说, 在 unification 的过程中, `auto p` 看到 `A` 被输入, 就马上去找答案了, 不会去管
`o` 后面给的 constraint. 怎么办呢?

很简单, 我们保证所有的 row predicate 都要在显式参数之后出现即可:

```agda
f : {A : row} -> (o : object A) -> {auto p : (n: number) <: A} -> number
f _ _ _ = 42

example : number
example = f {n: 69}
```

### 使用 Idris 2 写的 PoC 代码

我们用 `Nat` 替代 row, 可以用 Idris 2 写出满足我们 PoC 设计的代码:

```idr
data Row : Nat -> Type where
  Field : (n : Nat) -> Row n

f0 : {n : Nat}
  -> (r : Row n)
  -> {auto 0 _ : n > 2 = True}
  -> Nat
f0 {n} _ = n

-- 一般用法.
f1 : Nat
f1 = f0 (Field 3)

-- 试试 Sigma type 起不起效.
g0 : (n : Nat ** (p : n > 2 = True ** Row n))
g0 = (3 ** Refl ** Field 3)
```

### Implicit arguments 是怎么实现的?

Andras Kovacs 的 [elaboration-zoo] 项目对这个功能有大量的实践和解释, 在这里只讲一些大概.

RowScript 仅实现了 `03-holes` 和 `04-implicit-args` 两个例子中的算法的一部份.

简单来说, implicit 的设计需要基于 holes, 在它 [`Main.hs`] 的开头注释写的非常的清楚,
仔细阅读就能了解 holes 和 pattern unification 是怎么玩转起来的: 创建一个全局声明,
没有具体定义, 但是有具体类型 (即 constraint, expected type), 在后续计算中能得出 solution
的, 就给这个全局声明赋予定义, 如果解不出来:

* 是 elaborator 帮忙插入的 hole 的话, 报 *unsolved meta* 错误
* 是用户插入的 hole 的话, 提示用户记得填写

基于 hole 和 pattern unification 之后, implicit argument 实际上就是这个功能的扩展,
这个在 [`example.txt`] 有详细的规则解释, 大概就是:

* 遇到 application 时, `f` 的类型是函数 (下面用 `f_ty` 指代 `f` 的类型):
  * 如果是 `f x` (即 argument info 是 *explicit*, 函数调用是 *显式* 的), 并且 `f_ty`
    的 parameter info 是 *implicit* (即函数的参数类型是 *隐式*), 那么插入一个 hole, 即
    `((f ?y) x)`
  * 如果是 `f {x}`, 那啥也不用干, 说明用户强制输入了一个隐式参数
  * 如果是 `f {t = x}`, 则不断插入 hole, *直到* 有个形式参数叫做 `t` 的时候才不插入,
    没遇到就报错, 说 `t` 找不到

关于 lambda 语法的 *隐式形式参数* 插入规则 RowScript 没做, 这是因为在 RowScript 的语法中,
函数定义的 body 无论如何都能直接引用到隐式的参数, 所以这个功能是不必要的, 举个例子:

```agda
id : {t : Type} -> t -> t
id x = x
```

这个 `id` 在 Agda 里面, 如果想要在函数定义里引用到隐式形式参数 `t`, 必须加一个 `{t}`:

```agda
id : {t : Type} -> t -> t
id {t} x = x
```

而在 RowScript 是不需要的, 因为:

```ts
function id<T>(x: T): T {
    // 这里是可以出现 `T` 的, 直接用.
    return x;
}
```

函数的定义内部是可以直接引用到 `T` 的, 不需要额外的语法.

## 彩蛋: 在 RowScript 中实现 interface

以下的 Idris 2 代码将会是后续 interface 实现的 PoC 和理论基础, 在此进行一个剧透:

```idr
Additive : {t : Type} -> Type
Additive {t} =
  (  add : ((x : t ** (y : t ** Unit)) -> t)
  ** Unit
  )

NatAdditive : Type
NatAdditive = Additive {t = Nat}

NatAdd : (x : Nat ** (y : Nat ** Unit)) -> Nat
NatAdd (a ** (b ** _)) = a + b

example : Nat
example =
  let
    %hint
    inst : NatAdditive
    inst = (NatAdd ** ())
  in
    foo

  where
    foo : {auto a : NatAdditive} -> Nat
    foo {a = (add ** _)} = add (1 ** (2 ** ()))
```

[Idris 2 Miscellany]: https://idris2.readthedocs.io/en/latest/tutorial/miscellany.html#auto-implicit-arguments
[elaboration-zoo]: https://github.com/AndrasKovacs/elaboration-zoo
[`Main.hs`]: https://github.com/AndrasKovacs/elaboration-zoo/blob/4686b2538477e19b439906abc8a1f7b6eceebda5/03-holes/Main.hs#L23
[`example.txt`]: https://github.com/AndrasKovacs/elaboration-zoo/blob/4686b2538477e19b439906abc8a1f7b6eceebda5/04-implicit-args/example.txt#L32
