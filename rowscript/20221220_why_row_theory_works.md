# 为什么 Row Theory 能行得通?

* Anqur
* 2022-12-20

> *TODO*

```idr
data Row : Nat -> Type where
  Field : (n : Nat) -> Row n

f0 : {n : Nat}
  -> (r : Row n)
  -> {auto 0 _ : n > 2 = True}
  -> Nat
f0 {n} _ = n

f1 : Nat
f1 = f0 (Field 3)

g0 : (n : Nat ** (p : n > 2 = True ** Row n))
g0 = (3 ** Refl ** Field 3)
```

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
