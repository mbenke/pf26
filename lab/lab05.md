## IO

Napisz prostą funkcję interaktywną np. "chodzenie po mapie"
- program wyświetla współrzędne, użytkownik wpisuje jedno z poleceń `n e s w`, program wyswietla nowe współrzędne.
W którymś miejscu może być "skarb".

Uwaga: użyj `ghc` lub `runghc` raczej niż `ghci`, inaczej możesz napotkać błąd

```
<stdin>: hGetChar: illegal operation (handle is semi-closed)
```

- Powtórz poprzednie zadanie przy pomocy funkcji IO (`getChar`, `putStr`) itp.

- Skompiluj program przy użyciu `ghc`

- Stwórz prosty pakiet `cabal`
    - z samym modułem `Main`
    - dodaj moduł i zaimportuj z niego funkcję
    - dodaj zależnośc od pakietów `haskell-src` i `containers`
- Napisz program, który wypisze swoje argumenty

- Zapoznaj się z dokumentacją [haskell-src](https://hackage.haskell.org/package/haskell-src) i stwórz program wczytujący moduł w Haskellu i wypisujący nazwy zdefiniowanych w nim funkcji.

**Przydatne funkcje:**
``` haskell
getArgs :: IO [String]  -- System.Environment
readFile :: FilePath -> IO String
parseModule :: String -> ParseResult HsModule -- Language.Haskell.Parser
```
