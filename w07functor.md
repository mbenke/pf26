---
title: Programowanie Funkcyjne
subtitle: Wyższe klasy
author:  Marcin Benke
date: Wykład 7, 20.4.2026
---

# Functor

1. Motywacja - `map`
2. Klasa `Functor`
3. Klasy konstruktorowe

## Przypomnienie - map
Przypomnijmy funkcję `map` na listach:

``` haskell
map :: (a -> b) -> [a] -> [b]
map f [] = []
map f (x:xs) = f x : map f xs
```

Podobne funkcje możmy pisać dla innych struktur np. drzew

``` haskell
mapTree :: (a -> b) -> Tree a -> Tree b
mapTree f = go where
  go Empty = Empty
  go (Node x l r) = Node (f x) (go l) (go r)
```

Ogólna zasada: zachowujemy strukturę, a do elementów stosujemy podaną funkcję.

## Functor

``` haskell
map :: (a -> b) -> [a] -> [b]
mapTree :: (a -> b) -> Tree a -> Tree b
mapMaybe :: (a -> b) -> Maybe a -> Maybe b
```

Możemy wyrazić ten schemat przy pomocy klasy

``` haskell
class Functor t where
    fmap :: (a -> b) -> t a -> t b

instance Functor [] where
    fmap = map

instance Functor Tree where
    fmap = mapTree

instance Functor Maybe where
    fmap f Nothing = Nothing
    fmap f (Just x) = Just (f x)
```

NB ta klasa mogła by sie nazywać `Fun` albo `Mappable` i w niektórych językach tak się nazywa.

Haskell lubi straszyć noobów nazwami typu *Semigroup, Monoid, Functor, Monad, ...*
nie nalezy się tego bać.

### Konstruktory wartości i typów

```haskell
data Tree a = Leaf a | Branch (Tree a) (Tree a)

mapTree :: (a->b) -> Tree a -> Tree b
mapTree f (Leaf a) = Leaf (f a)
mapTree f (Branch l r) = Branch (m l) (m r) where
  m = mapTree f
```
**Leaf** jest 1-argumentowym konstruktorem,<br/>
**Branch** — 2-argumentowym.

Per analogiam mówimy, że **Tree** jest jednoargumentowym *konstruktorem typu*:

-   jeśli **x** jest wartością, to **Leaf x** jest wartością;
-   jesli **a** jest typem, to **Tree a** jest typem.

(ale nie na krzyż - **Leaf Int** ani **Tree 1** nie mają sensu)

### Klasy konstruktorowe

Typy polimorficzne jak **\[a\]** czy **Tree a** mogą być instancjami klas (przeważnie pod warunkiem, ze **a** jest też instancją odpowiedniej klasy)…

```haskell
    data Tree a = Leaf a | Branch (Tree a) (Tree a)
      deriving Show

    instance Eq a => Eq (Tree a) where
      Leaf x == Leaf y = x == y
      Branch l r == Branch l' r' = (l==l')&&(r==r')
      _ == _ = False
```

…ale są też klasy, których instancjami są nie typy, a *konstruktory typów*,
takie jak **Tree** czy **Maybe**.

### Functor jest klasą konstruktorową

Spójrzmy jeszcze raz na klasę `Functor` i jej instancje

``` haskell
class Functor t where
    fmap :: (a -> b) -> t a -> t b

instance Functor [] where

instance Functor Tree where

instance Functor Maybe where
```

`Tree` i `Maybe` nie są typami, ale konstruktorami typów.

Podobnie, w tym kontekście `[]` oznacza konstruktor typu list.

Klasa `Functor` jest klasą konstruktorową - jej instancjami są konstruktory typów.

Klasy typów opisują własności typów, klasy konstruktorowe - własności konstruktorów typów.<br />

Konstruktory typów i ich własności są pojęciami *wyższego rodzaju* (ang. *higher-kinded*)

### Prawa dla fmap

``` haskell
class Functor t where
    fmap :: (a -> b) -> t a -> t b
```

Prawa dla `fmap` są analogiczne jak dla `map`:

``` haskell
fmap id = id
fmap f . fmap g = fmap (f . g)
```

Pierwsze prawo sugeruje, że `fmap` zachowuje strukturę.<br/>
Zauważmy, że `fmap` jest polimorficzne, zatem z uwagi na parametryczność musi działać dla wszystkich funkcji tak samo.

**Pytanie:** czy `fmap f x` ma zawsze tyle samo elementów co `x`?

Tym niemniej `fmap` nie zawsze musi być tak trywialne jak dla list czy drzew.<br/>
Pomyślmy na przykład o zbiorach czy drzewach BST i funkcjach, które nie są różnowartościowe czy monotoniczne.

Dla typów, które są instancją `Monoid`, można pokusić się o inną charakteryzacje:

``` haskell
fmap f mempty = mempty
fmap f (x <> y) = fmap f x <> fmap f y
```

## Skróty

Dane drzewo, jak uzyskać drzewo tego samego kształtu, ale złożone z samych jedynek?

I podobnie dla list ...

``` haskell
allOnes :: Functor t => t a -> t Char
allOnes t = fmap (const '1') t
```

ale można krócej:

``` haskell
allOnes t = '1' <$ t

-- (<$) :: Functor f => a -> f b -> f a
-- (<$) = fmap . const
-- '1' <$ t = (<$) '1' t = (fmap . const) '1' t = fmap (const '1') t

-- >>> '1' <$ "abc"
-- "111"

-- >>> allOnes $ Node 2 (Node 1 Empty Empty) (Node 3 Empty Empty)
-- Node '1' (Node '1' Empty Empty) (Node '1' Empty Empty)
```

To może trochę słaby przykład, ale w przyszłości zobaczymy  lepsze, np.

``` haskell
pBool :: Parser Bool
pBool =  EBool True  <$ pKeyword "true"
     <|> EBool False <$ pKeyword "false"
```

## Applicative

Motywacja: chcemy napisać funkcję `addMaybe :: Maybe Int -> Maybe Int -> Maybe Int`

Po co? `Maybe` jest typowym rozwiązaniem dla obliczeń zawodnych.

Łatwo to zrobić przez dopasowanie, ale czy możemy skorzystać z `fmap`? (NB `<$>` to infiksowa wersja `fmap`)

``` haskell
ghci> fmap (+1) (Just 5::Maybe Int)
Just 6

ghci> (+) <$> (Just 2) <$> (Just 3::Maybe Int)

error: [GHC-83865]
    • Couldn't match expected type: Int -> b
                  with actual type: Maybe (a0 -> a0)
```

Tak się nie da :(

(NB wskazanie typu `::Maybe Int`) tylko po to, żeby klasa `Num` nie zaciemniała obrazu)

### Problem z typami

Co poszło nie tak?

``` haskell
ghci> :t (+1) <$> (Just 5::Maybe Int)
(+1) <$> (Just 5::Maybe Int) :: Maybe Int

ghci> :t (+) <$> (Just 5::Maybe Int)
(+) <$> (Just 5::Maybe Int) :: Maybe (Int -> Int)
```

Potrzebujemy albo funkcji

``` haskell
fmap2 :: (a -> b ->c) -> Maybe a -> Maybe b -> Maybe c
```
(tudzież `fmap3,fmap4`, itd.) ...albo funkcji:

``` haskell
apply :: Maybe (a -> b) -> Maybe a -> Maybe b
```
wtedy
``` haskell
ghci> apply((+) <$> (Just 2)) (Just 3::Maybe Int)
Just 5
```

Początkowo stosowano pierwsze podejście: funkcje `liftM .. liftM5`, ale to mało elastyczne.

### Lift me to the stars

drugie podejście jest lepsze, `<$>` (fmap) i `<*>` (apply) pozwalają wyrazić `fmap_n` dla n>0:

``` haskell
(<*>) ::  Maybe (a -> b) -> Maybe a -> Maybe b

fmap2 :: (a -> b ->c) -> Maybe a -> Maybe b -> Maybe c
fmap2 f a b = f <$> a <*> b
fmap3 f a b c = f <$> a <*> b <*> c
...
```

Czyli  pozostaje n=0;
dla `Maybe` można użyć `Just`, ale chcemy stworzyć ogólny mechanizm:

``` haskell
class Functor f => Applicative f where
    pure :: a -> f a
    (<*>) :: f (a -> b) -> f a -> f b
```

Także `<$>` (czyli `fmap`) da się wyrazić:

``` haskell
f <$> a = pure f <*> a
```

W rezultacie

``` haskell
ghci> pure (+) <*> Just 2 <*> Just 3
Just 5
```

### Bro, do U even liftA2 ?

Funkcje dwuargumentowe są na tyle powszechne, że `Applicative` ma odpowiednik `fmap2`:

``` haskell
liftA2 :: Applicative f => (a -> b -> c) -> f a -> f b -> f c
liftA2 f x = (<*>) (fmap f x)

(<*>) :: f (a -> b) -> f a -> f b
(<*>) = liftA2 id
```

![](https://i.kym-cdn.com/entries/icons/mobile/000/009/740/DoULift.jpg
)

### Implementacja Applicative dla Maybe

``` haskell
instance Applicative Maybe where
    -- pure :: a -> Maybe a
    pure = Just
    -- (<*>) :: Maybe (a -> b) -> Maybe a -> Maybe b
    Just g <*> Just x = Just (g x)
    _      <*> _      = Nothing

```


### Programowanie z efektami

Zaczęliśmy od uogólnienia `map` na inne kolekcje i funkcje wieloargumentowe.

Ważniejszą motywacją jest jednak programowanie z efektami; używaliśmy przykładu **Maybe**<br />
gdzie "efektem" jest porażka,
ale podobny charakter mają inne rodzaje efektów, np.

``` haskell
ghci> :t getLine
getLine :: IO String
ghci> (++) <$> getLine <*> getLine    -- albo: liftA2 (++) getLine getLine
Ala
bama
"Alabama"
```
Biblioteka standardowa zawiera funkcję sekwencjonującą obliczenia

``` haskell
sequenceA :: Applicative f => [f a] -> f [a]
sequenceA [] = pure []
sequenceA (x:xs) = pure (:) <*> x <*> sequenceA xs
```

Na kolejnych wykładach poznamy bardziej ogólne mechanizmy.

### Na skróty
Czasami przy łączeniu efektów `<*>` interesuje nas tylko jeden z wyników (ale oba efekty)

``` haskell
const <$> program <*> eof
```
Wynik `eof` jest nieciekawy natomiast interesuje nas wynik `program`

``` haskell
ghci> const <$> putStr "Ala" <*> putStrLn "bama"
Alabama
```
I
``` haskell
ghci> flip const <$> putStr "Enter something: " <*> getLine
Enter something: 42
"42"
```
tu z kolei wynik `putStr` jest nieciekawy.

Możemy to zapisać prościej przy uzyciu skrótów

``` haskell
program <* eof
putStr "Enter something: " *> getLine
```
zdefiniowanych w klasie **Applicative** jako

```
(*>) :: Applicative f => f a -> f b -> f b
a1 *> a2 = (id <$ a1) <*> a2

-- (<$) = fmap . const
-- a1 *> a2 = (id <$ a1) <*> a2 = ((const id) <$> a1) <*> a2 = (flip const <$> a1) <*> a2
-- const id x y = id y = y = flip const x y

(<*) :: Applicative f => f a -> f b -> f a
(<*) = liftA2 const
```

`*>` jest podobne do `>>`, ale znacznie ogólniejsze.

### Either

**Maybe** daje tylko informacje czy obliczenie się powiodło czy nie.<br/>
Możemy dodać komunikaty o błędach przy pomocy **Either**:

``` haskell
data Either error result = Left error | Right result
```

**Either** jest dwuargumentowym konstruktorem typu,<br/>
ale po ustaleniu pierwszego argumentu `(Either e)` jest jednoargumentowym

``` haskell
instance Functor (Either e) where
  fmap _ (Left e) = Left e
  -- Uwaga: "Left e" ma różne typy po lewej i po prawej, nie można `fmap f x@(Left e) = x`
  fmap f (Right r) = Right (f r)

instance Applicative (Either e) where
  pure = Right
  -- pure a = Right a
  (Right f) <*> (Right x) = Right (f x)
  (Left e)  <*>  _        = Left e      -- uwaga jak przy Functor
  _         <*> (Left e)  = Left e

```

### Either odwrotnie? (Rehtie?)

``` haskell
instance Functor (Either e) where
  fmap _ (Left e) = Left e
  fmap f (Right r) = Right (f r)
```

A co jeśli chcielibysmy odwrócić znaczenie Left i Right?

``` haskell
-- Uwaga: to nie jest poprawny Haskell

instance Functor (Either ? e) where
  fmap _ (Right e) = Right e
  fmap f (Left r) = Left (f r)
```

tak niestety nie można...

### Ey, what about the ring trick?

<img src="data:image/jpeg;base64,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" width=800>


### The ~~ring~~ Flip trick

Przez analogię do funkcji

``` haskell
flip f x y = flip f y x
```

możemy zdefiniować "funktor wyższego rzędu":

``` haskell
newtype Flip f a b = Flip { unFlip :: (f b a) }
  deriving (Eq, Show)

instance Functor (Flip Either a) where
  fmap f (Flip (Right e)) = Flip (Right e)
  fmap f (Flip (Left a)) = Flip (Left (f a))


-- >>> fmap succ (Flip (Left 'x'))
-- Flip {unFlip = Left 'y'}

pamf f = unFlip . fmap f . Flip

-- >>> pamf succ (Left 'x')
-- Left 'y'
```

**Ćwiczenie:** spróbuj napisać typ funkcji `pamf`

### That's the ring trick

![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgkEts7w_B9dxmokq5M8yFOS0KgGZPrLRoDlCeuescCPs3hg5veAlnet6CldcHrwXgxnhA2BKo4B5ODHNo6lBqapSvzyZAX98Ur1LAvq1FlFEBtbm6dYtZuBt9cwNm_ziqthBCrcs9mcWY/s1600/pierscionek.png)


### Pary

W przypadku Either nie miało to może sensu, ale co z parami? łatwo zastosowac funkcję do drugiego elementu:

``` haskell
data Pair a b = Pair a b

instance Functor (Pair a) where
  fmap f (Pair a b) = Pair a (f b)

-- >>> fmap succ (1,'x')
-- (1,'y')
```

Dla pierwszego elementu mozemy posłuzyć się sztuczką z `Flip`:

``` haskell
instance Functor (Flip Pair a) where
  fmap f (Flip (Pair a b)) = Flip (Pair (f a) b)

instance Functor (Flip (,) a) where
  fmap f (Flip (a, b)) = Flip ((f a), b)

-- pamf f = unFlip . fmap f . Flip

-- >>> pamf succ ('x','a')
-- (y','a')
```

NB `pamf` jest uniwersalne

### Wiele wyników

Czysta funkcja daje dokładnie jeden wynik dla danego argumentu.<br/>
Obliczenia zawodne czasem nie dają wyniku, dlatego używamy `Maybe`.

A co z obliczeniami, które moga dać wiele wyników ("niedeterminizm")?<br/>
Możemy użyć list (chociaż są lepsze rozwiązania):

``` haskell
instance Applicative [] where
    -- pure :: a -> [a]
    pure x = [x]
    -- (<*>) :: [a -> b] -> [a] -> [b]
    gs <*> xs = [g x | g <- gs, x <- xs]
```

Intuicja: obliczenie dla funkcji daje listę możliwości, podobnie obliczenie dla argumentu;<br/> obliczenie dla aplikacji da wszystkie możliwe kombinacje:

``` haskell
ghci> [(+1), (*2)] <*> [5, 7]
[6,8,10,14]
ghci> pure (+1) <*> [1,2,3]
[2,3,4]
ghci> pure (+1) <*> pure 6 :: [Int]
[7]
ghci> [(+1), (*2)] <*> pure 6
[7,12]
```

### Uogólnienie: klasa **Alternative**

``` haskell
class Applicative f => Alternative f where
  empty :: f a
  (<|>) :: f a -> f a -> f a
  some :: f a -> f [a]         -- v+
  some v = (:) <$> v <*> many v
  many :: f a -> f [a]         -- v*
  many v = some v <|> pure []

instance Alternative [] where
  empty = []
  (<|>) = (++)

instance Alternative Maybe where
  empty = Nothing
  Nothing <|> r = r
  l       <|> _ = l
```

przykłady użycia **Alternative** zobaczymy przy omawianiu zagadnień parsingu.



# Bonus

Pytania?

## Tour de force - palindromy
``` haskell
ghci> palindrome = (==) <*> reverse
ghci> palindrome "ala"
True
ghci> palindrome "ela"
False
ghci> palindrome "kajak"
True
```
## Tour de force - palindromy

<img src="https://imgur.com/FCgfKjB.png" width=1600 height=900>

### Funkcje jako Applicative

Konstruktor typu funkcji `(->)` jest dwuargumentowy<br/>
Ustalając dziedzinę możemy uzyskac jednoargumentowy: `((->) r)`

``` haskell
instance Applicative ((->) r) where
  pure x y = x
  (<*>) f g z = f z(g z)
```

Pamiętacie kombinatory?

```
K x y = x
S f g z = f z(g z)
```

funkcje są strukturą aplikatywną

## Prawa dla Applicative

``` haskell
pure id <*> x  =  x                              -- identity
pure (g x)     =  pure g <*> pure x              -- homomorphism
x <*> pure y   =  pure (\g -> g y) <*> x         -- interchange
x <*> (y <*> z) = (pure (.) <*> x <*> y) <*> z   -- composition
```

Te prawa nie są tak groźne jak wyglądają:

1. Pierwsze znamy już dla `fmap` (`pure id <*> x = fmap id x = x`)
2. Drugie mówi że `pure` zachowuje aplikację
3. Trzecie mówi, że obliczenia czyste można wykonać przed albo po obliczeniu z efektami
4. Czwarte zaraz wyjaśnimy

Prawa te formalizują intuicję, że `pure` reprezentuje obliczenie czyste (bez efektów).

Zapewniają także, że każde wyrażenie aplikatywne da się zapisać w postaci

``` haskell
pure g <*> x1 <*> ... <*> xn

```
czyli
``` haskell
g <$> x1 <*> ... <*> xn
```

### Prawa dla Applicative - identity

``` haskell
pure id <*> x  =  x                              -- identity
```

To prawo jest odpowiednikiem pierwszego prawa dla `fmap`:

``` haskell
fmap id x = x
```

i podobnie jak ono mówi o zachowywaniu struktury (`pure f <*> x = fmap f x`)

### Prawa dla Applicative - composition

``` haskell
x <*> (y <*> z) = (pure (.) <*> x <*> y) <*> z   -- composition
```

Pamiętając, że `<*>` uogólnia aplikację, to prawo uogólnia własność złożenia funkcji

``` haskell
x (y  z) = (.) x y z
```

Na przykład

``` haskell
ghci> (.) (+1) (*2) 3
7
ghci> (+1) ((*2) 3)
7

ghci> (pure (.) <*> [(+1)] <*> [(*2)]) <*> [1,2,3]
[3,5,7]
ghci> [(+1)] <*> ([(*2)] <*> [1,2,3])
[3,5,7]
```

### Prawa dla Applicative - homomorphism

Homomorfizm to mapowanie algebr zachowujące ich strukturę, na przykład dla słów z konkatenacją</br>
(albo dowolnego monoidu)


$$ f(\epsilon) = \epsilon $$
$$ f(u\cdot v) = f(u)\cdot f(v) $$

<!--
f(ε) = ε

f(uv) = f(u)f(v)
-->

``` haskell
pure id <*> x  =  x                              -- identity
pure (g x)     =  pure g <*> pure x              -- homomorphism
```

Przy czym tutaj zmienia się nośnik i operacja. Odpowiednikiem dla słów byłby homomorfizm

$$h : \langle \Sigma^*, \epsilon, \cdot\rangle \to \langle N, 0, +\rangle $$
$$ h(\epsilon) = 0 $$
$$ h(u\cdot v) = h(u) + h(v) $$


### Prawa dla Applicative - interchange

``` haskell
x <*> pure y   =  pure (\g -> g y) <*> x         -- interchange
```

To prawo mówi, że obliczenia czyste można wykonać przed albo po obliczeniu z efektami

### Solo: A Star Wars Story

Haskell nie ma krotek jednoelementowych, ale możemy zdefiniować (patrz `Data.Tuple`)

```haskell
data Solo a = MkSolo a

instance Functor Solo where
    fmap f (MkSolo a) = MkSolo (f a)

instance Applicative Solo where
    pure = MkSolo
    MkSolo f <*> MkSolo x = MkSolo (f x)
```

**Ćwiczenie:** sprawdź, że prawa `Applicative` zachodzą dla `Solo`.



### Tour de force - soczewki

Chcemy zdefiniować typ pozwalający skupić się na pewnym fragmencie struktury danych.

Nazwijmy go "soczewką":

``` haskell
type Lens a b
view :: Lens a b -> a -> b                -- odczytaj pole
set  :: Lens a b -> b -> a -> a           -- ustaw pole
over :: Lens a b -> (b -> b) -> (a -> a)  -- zmodyfikuj pole
```


Znów przydadzą się funktory wyższego rzędu:

``` haskell
newtype I x = I { unI :: x }

instance Functor I where
  fmap f  = I . f . unI

newtype K b x = K { unK :: b }

instance Functor (K b) where
  fmap f (K b) = K b


type Lens a b = forall t . Functor t => (b -> t b) -> (a -> t a)

view :: Lens a b -> a -> b
view l a = unK (l K a)

over :: Lens a b -> (b -> b) -> (a -> a)
over l f = unI . l (I . f)
-- over l f a = unI $ l f' a where f' b = I (f b)

set :: Lens a b -> b -> a -> a
set l x = over l (const x)
```

### Tour de force: `fmap fmap fmap`

``` haskell
ghci>  fmap fmap fmap negate (+) 2 3
-5
```

WTF?

### Tour de force: `fmap fmap fmap` explained

``` haskell
instance Functor ((->) r) where
  fmap = (.)

fmap fmap fmap = fmap . fmap


(f . g) x = f(g x)

fmap fmap fmap negate (+) 2 3
= (fmap . fmap) negate (+) 2 3
= ((fmap . fmap) negate) (+) 2 3
= fmap (fmap negate) (+) 2 3
= (fmap (fmap negate) (\x -> (\y -> x + y))) 2 3
= (fmap negate . (\x -> (\y -> x + y))) 2 3
= (fmap negate((+) 2 )) 3
= fmap negate (+2) 3
= (negate . (+2)) 3
= negate ((+2) 3)
= negate 5
= -5
```

``` haskell
ghci>  fmap fmap fmap negate (+) 2 3
-5
```
