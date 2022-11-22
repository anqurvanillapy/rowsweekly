# RowScript 现状和路线

> * Anqur
> * 2022-11-22

## 啥事 RowScript?

根据 40 哥哥的描述:

> RowScript is a robustly-typed functional language that compiles to efficient
> and reliable JavaScript.

这句话是我们整个语言的 slogan/propaganda (大雾), 但其实也是终极目标: 我们希望的是能用较新的
PLT ideas 去让 JavaScript 更加 consistent, 更加 scalable.

稍微 cringe 一点的描述 (锤哥警觉) 会是:

> RowScript is a purely functional programming language with imperative syntax.

我们会沿用 JavaScript/TypeScript 的语法, 而不是 Haskell family 或 ML family 的风格,
这样在项目初期可以沿用它们的高亮插件, 并且用户会更容易上手一些.

我们的目标群体会是喜欢在 imperative 语言里写少量 functional 的那些人 (比如我),
并且我更希望 **尽量避免使用学术用词**, 尽可能沿用一些能同时符合工业和学术的术语,
内部文档的话则可以大量使用 (但要标注好之间的联系什么的), 原则就是: 用户需要一些操作上的成本,
比如二次确认 (文本折叠), 比如没有 shortcut 的比较深的目录, 才能看到大量的学术用词和材料引用,
确保他们主动地想去了解再展示.

比如, 下面的内容我们将具体讨论 *purely functional* 的含义, 并且确定此刻需要讨论更多学术用语.

<details>
<summary>那么请点击此处打开内容</summary>
<p>没错, 你学会了捏.</p>
</details>

## 语言内核

> ⚠️ 注意! 下面的内容我们将大量使用学术用语!

从学术来讲, 认真的说, RowScript 是一门这样的语言:

> RowScript is a dependently-typed programming language extended with
> record-concatenation-based row polymorphism.

我们并不考虑 TypeScript 那种 intersection types 的变种, 也不考虑 Haskell 那样的 system
F 变种, 这是考虑到用户的一些被 TS 搞得 **拧巴** 但是又 **顽固** 的使用习惯:

* Debug 体验较差且可读性不高的 *类型体操*
* 基于类型体操, 大量去做一些 type-level 计算, 比如 API 的 validator 规则, 比如 ORM 声明

并且, TypeScript 本身有一些比较让我诟病的玩意:

* 一些奇怪的 properties, 比如 `readonly`, 比如指代 optional 的 `?`
* Union types 真的不好用

这些要解决的问题, 我们希望通过如下语言特性来去解决:

* 对于类型体操, 我们直接引入 **dependent types**, 但是在语法上要进行封装和限制,
  体验上尽量接近 system F, 并且报错时可以学习 Lean 4, 不要猛吐 core terms 啥的
* 分开哪些 function definition 可以直接 normalize 到底 (type-level 计算), 哪些直接
  leave untouched (正常 codegen 到 JS 的模式)
* 支持 **FBIP** (functional but in-place), 这点是基于 Perceus 和 FLRA 来做的,
  可以看帆帆做的总结和讨论 [Summarized Document of Perseus Paper], 这样可以解决 codegen
  出的 JS 的性能问题
* 支持 record-concatenation-based **row polymorphism**, 这点基于 [AEDT] 来做, 解决
  JS 本身海量存量代码的 typechecking 问题, 我相信 row polymorphism 的表现能力会比较好些
* **Algebraic effect**, 目前想的不是很明白 (algebraic effect 的类型系统貌似还是个 open
  problem), 目前想法是 type-directed CPS transformation 加上 [ECMTT], 这个解决 JS
  框架状态转移相关代码的表达能力的问题
* 支持基于 proof search 的 **typeclass**, 目前还没想好具体方案, 但基于 proof search
  的机制很成熟 (比如 Idris), 这个特性主要提供更好的代码组织方式
* 做 **datatype generics**, 这个 Haskell 已经玩的很花了, 解决的是 ORM/validator
  等框架的 auto-deriving, 以及 deriving 逻辑的扩展, 这个特性还能用来写 JSON/YAML
  等的高性能 parser, 会比较常用

[Summarized Document of Perseus Paper]: https://discourse.rowscript-lang.org/t/summarized-document-of-perseus-paper/19
[AEDT]: https://github.com/rowscript/rowscript/issues/15
[ECMTT]: https://github.com/rowscript/rowscript/issues/15

## 现状

目前正在实现 RowScript 基础的 MLTT 能力, 做一个简单的 elaborator, 主要提供的东西:

* Universe, 并且是 type-in-type
* Pi 类型 (函数)
  * Lambda abstraction, lambda application
* Sigma 类型 (二元组)
  * Pair constructor, pair let-binding
* Unit 类型
  * Unit constructor, unit let-binding
* 其他简单类型: boolean, JS 的基础类型 `number`, `string`, `BigInt` 等

比如一个简单的 TypeScript 函数:

```ts
function foo(a: number, b: number): number { return a }
```

它会被 typecheck 成这个东西:

```plaintext
foo : (_tupled: (a: number) * (b: number) * unit) -> number
foo = lambda a: a
```

也就是说这个函数只会接收 **1 个** 参数, 参数类型是一个 Sigma, 连接 `a` 和 `b` 后以 `unit`
类型作为结尾. 我们用这种方式来 encode 一个基础的 JS 函数的声明和调用.

暂时不考虑的东西:

* Identity type, 等到时候出现类型体操很紧急需要这个玩意, 我们再去做就行了, 不难
* Universe level, 后续可以做 McBride 风格的, 但是我觉得需要 lift 到下一个 level 的话,
  这类型体操已经稍微有点过了 (

整个 elaborator 的流程:

1. Parser: 解析 surface syntax
   * 这个是由 Pest 来做的, 能声明好语法就行了
1. Translator: 将 surface syntax 翻译成 concrete syntax (expression),
  因为前者是一组一组的 pair, 能带的信息很少, 最好转换一层我们关心的树
1. Resolver: 给每个 reference 创建 `Rc` (没错, 我们用的是 capture-avoided
   substitution 来做替换, 没用 de Bruijn index 和 de Bruijn level,
   每一个变量名的唯一标识就是它的指针地址!), 然后检查看有没有 not-in-scope 的变量
1. Elaborator: 对着 concrete syntax 进行类型检查
   * Check: 类型检查, 已经知道期盼的类型, 直接对着 expression 怼
   * Infer: 类型推导, 我们把 expression 的类型推导出来, 看它的类型是不是和期盼的类型一致
1. 在 elaboration 期间, 会发生:
   * Normalize: 将 concrete syntax (expression) 转换成 abstract syntax (terms),
     这个时候 source file information 就全都丢失了
   * Unify: 判断两个 term 是不是同一个, 这个在 infer 就会用到
   * Rename: 把一个 term 里面的所有变量声明和 reference 重新生成, 刷新一下, 比如说
     function definition 它的签名里头有一套我们已经申请过 `Rc` 的变量, 但是通过它创建一个
     lambda abstraction 的时候不能沿用这个变量, 因为 signature 是个模板, 不能引用到
     signature 的变量声明里去

## 路线

做完 MLTT 的下一步可以并行做 row polymorphism 和 codegen, 然后就可以写一些和 JS 有点
interoperable 的代码了.

其他更多的东西暂时先不想.
