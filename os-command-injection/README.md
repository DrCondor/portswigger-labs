# OS command injection (shell injection)

> Aplikacja buduje **string polecenia powłoki** i wkleja do niego twój input, po czym uruchamia go przez shell. Jeśli wstrzykniesz **metaznak powłoki** (`; & | \n $() \``), twój input przestaje być „argumentem" i staje się **nowym poleceniem**. Skutek: wykonanie dowolnego kodu na serwerze — pełna kompromitacja.

---

## O co chodzi (koncept)

Aplikacja czasem woła narzędzie systemowe (ping, konwersja pliku, raport, legacy skrypt) i skleja polecenie z twoim inputem:

```
stockreport.pl <productId> <storeId>
```

Jeśli to polecenie idzie **przez powłokę** (`/bin/sh -c "..."`), shell interpretuje metaznaki. Twój input, wstawiony do stringa, może wtedy zawierać składnię, którą shell wykona jako osobne polecenie.

Po railsowemu — dokładnie to, przed czym Rails ostrzega:
```ruby
system("stockreport.pl #{product_id} #{store_id}")   # ❌ podatne (string → powłoka)
`ping #{host}`                                         # ❌ to samo
system("stockreport.pl", product_id, store_id)        # ✅ argv, bez powłoki
```

---

## Jak to działa (mechanizm)

### Metaznaki-separatory

Gdy polecenie idzie przez powłokę, shell interpretuje:

| Znak | Działanie |
|---|---|
| `;` | wykonaj kolejne polecenie (Linux) |
| `&` | wykonaj w tle / rozdziel polecenia |
| `\|` | pipe: stdout lewego → stdin prawego |
| `` `cmd` `` / `$(cmd)` | podstawienie wyniku polecenia |
| `\n` | nowa linia = nowe polecenie |

Przykład: input `& echo aiwefwlguh &` w `productID` daje `stockreport.pl & echo aiwefwlguh & 29` → trzy osobne polecenia. `echo` udowadnia wykonanie (twój string wraca w outpucie), końcowe `&` odcina to, co leci po injection.

### ⭐ Separator shella vs warstwa transportu

Najważniejsza praktyczna lekcja: payload musi najpierw **przeżyć parser protokołu** (form-urlencoded, JSON, URL), a dopiero potem interpretuje go shell. Znak może mieć **dwa znaczenia naraz**:
- `&` dla powłoki = separator poleceń,
- `&` dla parsera `x-www-form-urlencoded` = separator **parametrów** — a parser czyta ciało **pierwszy**.

Jeśli użyjesz `&` w ciele formularza, parser rozbije payload na pola, zanim cokolwiek dotrze do shella. Dlatego albo użyj znaku, który nie koliduje (`|`), albo **URL-zakoduj** separatory (`&`→`%26`).

To **execute, nie read** (ponad path traversal, który tylko czyta) — najgroźniejszy prymityw server-side.

---

## Ogólny schemat ataku

**Znajdź parametr lądujący w poleceniu → wstrzyknij separator shella + sondę → upewnij się, że payload przeżył warstwę transportu.**

Sondy:
- `whoami` — kto (nazwa usera),
- `echo <losowy string>` — potwierdza wykonanie (twój string wraca w outpucie).

Gdy outputu nie widać (**blind**): detekcja czasem (`& sleep 10 &` → mierzysz opóźnienie), kanałem out-of-band / DNS (Collaborator), albo przekierowaniem outputu do pliku w web rootcie.

---

## Jak się przed tym bronić

- **Nie buduj polecenia przez konkatenację danych.** Przekaż argumenty jako **tablicę** (`system("cmd", arg1, arg2)`) — to omija powłokę, więc metaznaki zostają danymi.
- Jeszcze lepiej: użyj biblioteki/API zamiast wołania shella.
- Jeśli naprawdę musisz do powłoki — rygorystyczna whitelista dozwolonych wartości.

---

## Słowniczek

- **OS command injection / shell injection** — user input w stringu polecenia uruchamianego przez powłokę; metaznaki → wykonanie dowolnych poleceń.
- **Shell metacharacters** — `; & | \n $() \``; separatory i podstawienia interpretowane przez powłokę.
- **In-band / returned injection** — output komendy wraca wprost w odpowiedzi HTTP. Przeciwieństwo: **blind**.
- **Warstwa transportu vs shell** — payload przechodzi przez parser protokołu ZANIM dotrze do powłoki; kolizja znaczeń znaku (`&`) psuje atak.
- **Argv vs shell** — argumenty jako tablica omijają powłokę (bezpieczne); interpolacja do jednego stringa — nie.

---

## Laby w tym temacie

- [01 — simple case](01-simple-case.md) — stock checker, `storeId=1|whoami`, output in-band. Pułapka: `&` kontra form-urlencoding.

*(PortSwigger ma dalej blind — time-based, out-of-band — i obejścia filtrów.)*

---

## Mindset / co się przenosi

Sygnał „to może być command injection": aplikacja robi coś, co pachnie **wywołaniem narzędzia systemowego** z twoim inputem w argumentach. OS command injection to **injection** — ta sama idea „przeskocz z danych do kodu" co SQLi (`'`) i XSS (`<script>`), tu wytrychem jest metaznak powłoki.

Rdzeń niezmienny: nigdy nie buduj polecenia powłoki przez konkatenację danych od użytkownika.
