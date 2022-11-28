# Row Polymorphism 前置设计

* Anqur
* 2022-11-28

## RowScript 的 Row Polymorphism 会长啥样?

一些 AEDT 论文里的内容咱在这里就不赘述了, 直接上 surface syntax 感受一下 AEDT 里的例子.

```ts
// RowScript 会用单引号来标注 row identifier, 这里为了高亮改用美元符号.
function getNum<$A, $B, $C>(a: {$A}, b: {$B}): number
    where $C = $A + $B
      and (n: number) in $C
      // RowScript 会用小于号表示 row containment, 这里为了高亮改用 in.
{
    return // ...
}
```

这个函数的意思是:

* 接收两个 object (就是一般的 JavaScript object), 返回一个 `number` 类型的数值
* 我期盼这两个 object 至少有一个有 `(n: number)` 这个成员, 但不在乎具体来自哪个
* 一个可能的函数实现就是, 把这个 `n` 成员的值给取出来, 在此忽略具体代码

谈到 RowScript 的类型检查, 为了方便, 我们暂时认为函数是柯里化的, 那么这个函数的检查结果:

```agda
getNum
    : {$A : Row} → {$B : Row} → {$C : Row}
    → {auto p : $C ≡ $A ⨀ $B}
    → {auto q : (n : number) ≤ $C}
    → (a : {$A}) → (b : {$B})
    → number
getNum = {??}
```

那么这个时候, 如果我们计算如下表达式:

```plaintext
getNum {n: 42} {m: 69} //=> 应返回 42
```

很多问题就会开始浮现...

## Implicit???

是的, 对于类型参数, 目测我们可以让用户主动指定类型 (也就是 type application, 例如
`foo<string>(42)`), 但是对于 row 参数, 傻傻的让用户去填写所有的参数非常的不友好. 这个时候,
只能是让 implicit argument 派上用场了.

首先, 用户调用 `getNum` 的时候并没有指定 row 参数, 即:

```plaintext
getNum {n: 42} {m: 69}
```

那么我们在类型检查的时候, 我们会有很多的 goals:

```plaintext
Current goals:

    $A : Row := ?
    $B : Row := ?
    $C : Row := ?
    p : $C ≡ $A ⨀ $B := ?
    q : (n : number) ≤ $C := ?

While typechecking:

    getNum {n: 42} {m: 69}
```

## 解谜嘛, 这个容易

我们作为人类, 其实填这些 goals 非常容易:

* 从 `{n: 42}` 可以得出, `$A` 的结果应该是 `(n: number)` (注意类型是 `Row` 不是 object)
* 从 `{m: 69}` 可以得出, `$B` 的结果应该是 `(m: number)`
* 从 `$C ≡ $A ⨀ $B` 可以得出, `$C` 的结果应该是 `(n: number, m: number)`
* 剩下的 `p` 和 `q` 从它们的 data constructor 直接构造, 即 `RowSat` (satisfied,
  条件满足)

```plaintext
All goals solved:

    $A : Row := (n: number)
    $B : Row := (m: number)
    $C : Row := (n: number, m: number)
    p : $C ≡ $A ⨀ $B := RowSat
    q : (n : number) ≤ $C := RowSat

While typechecking:

    getNum {n: 42} {m: 69}
```

## 但计算机不是人类

我们的类型检查器需要一定的 proof searching 算法优先级, 不能随便乱填; 学习 Idris 的做法,
对于上面的例子, 一个正确的查找优先级应该是:

1. 优先通过用户填的参数里更深的参数去反推, 比如 `$A` 的推理过程, 是从 `{$A}` 反推出来的
1. 从等式中推理, 比如 `$C` 的推理过程
1. 从 data constructor 直接构造, 比如 `p` 和 `q` 就可以直接构造并直接填, 但要注意的是在
   Idris 里面我记得要加上 `auto` 才能从构造器中去直接造出来, 所以上文中 `p` 和 `q` 做了标记

目前整个算法流程大致是这个样子, 不知道实际实现起来会遇到什么问题, 不知道要不要去看 Agda 和
Idris 的代码...
