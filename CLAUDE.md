# appsec-portswigger — notatki z nauki AppSec

## Co to za miejsce

Repo z **notatkami z nauki cybersecurity (AppSec)** na bazie labów **PortSwigger Web Security Academy**.
Kontekst osoby uczącej się: **wieloletni dev Ruby on Rails**, który zaczyna grzebać w appsecu jako nowy kierunek (docelowo m.in. w stronę BSCP). Traktuj mnie jak doświadczonego programistę, ale **nowicjusza w security** — tłumacz mechanizmy, nie same kroki.

## Jak pracujemy

- Robię labki **po jednej**, na żywo — analizujemy razem *co* i *jak* zrobić.
- **Będę Cię dużo pytać "o co tu chodzi"** — Twoja rola to przeprowadzić mnie przez nowy dla mnie obszar: wyjaśniać koncepty, mechanizmy, pułapki, a nie tylko podawać rozwiązanie.
- Po (albo w trakcie) każdej labki powstaje **jedna notatka** = jeden plik `.md`. To jest baza wiedzy, do której wracam, żeby sobie coś przypomnieć.
- Notatki piszemy **po polsku**.

## Struktura repo

- Jeden **katalog = jeden temat** PortSwiggera (np. `web-cache-deception/`).
- Wewnątrz katalogu: `NN-krotka-nazwa.md` (numer = kolejność robienia labki w danym temacie).
- Kolejne tematy dostają własne katalogi obok.

```
appsec-portswigger/
├── CLAUDE.md
└── web-cache-deception/
    ├── 01-path-mapping.md
    └── 02-path-delimiters.md
```

## Format notatki (trzymamy się tego szablonu)

Każdy plik labki zawiera, mniej więcej w tej kolejności:

1. **Nagłówek z metadanymi** — kategoria, poziom (Apprentice/Practitioner/Expert), data, status (✅ rozwiązany), narzędzia.
2. **Cel labu** — jedno zdanie, co trzeba osiągnąć.
3. **Koncept** — o co chodzi w tej klasie podatności (prostym językiem, z analogiami).
4. **Mechanizm tego labu** — konkretny trik / payload i *dlaczego* działa.
5. **Kroki / przepływ ataku** — krok po kroku; mile widziany diagram `mermaid` przepływu.
6. **Pułapki** — na czym realnie można się przejechać (z przykładami z rozwiązywania).
7. **Pojęcia do zapamiętania** — słowniczek terminów.
8. **Mindset** — co się przenosi na inne laby / co się powtarza.

Styl: konkret, tabelki do porównań (origin vs cache, lab X vs lab Y), **pogrubienia** na sednie, dużo "dlaczego to działa", a nie tylko "co kliknąć.

## Dotychczasowe tematy

### `web-cache-deception/` — Web Cache Deception
Sedno całej klasy: **rozbieżność między tym, jak URL interpretuje origin server, a jak interpretuje cache**. Origin oddaje prywatną stronę (bo widzi swój endpoint), cache ją zapisuje (bo widzi "statyczny plik"). Ofiara odwiedza spreparowany URL swoją sesją → jej prywatne dane lądują we wspólnym cache → atakujący je odczytuje.

- **01 — path mapping**: origin mapuje po prefiksie i ignoruje sufiks (`/my-account/wcd.js`).
- **02 — path delimiters**: origin ucina ścieżkę na znaku-delimiterze (`/my-account;wcd.js`, np. `;` w Spring); szukanie delimitera Burp Intruderem.
- **03 — origin server normalization**: origin normalizuje traversal, cache nie → `/resources/..%2fmy-account` (encoded slash `%2f`, static directory rule na `/resources`).
- **04 — cache server normalization**: cache normalizuje, origin nie + delimiter → `/my-account%23%2f..%2fresources%2fwcd` (`#` jako delimiter wysyłany jako `%23`). Dużo debugowania *dostarczenia* (Store, timing, składnia exploita).

Temat **Web cache deception domknięty** (labki 01–04).

Powtarzalny schemat ataku: **znajdź rozbieżność → dostarcz link ofierze (exploit server) → czytaj dane z cache** (`X-Cache: miss/hit`).

### `web-llm-attacks/` — Web LLM attacks
Aplikacja podpina LLM-a do backendu i daje mu dostęp do **funkcji/narzędzi** (function calling). LLM to most między tekstem użytkownika a realnymi funkcjami backendu. Podatność: model ma dostęp do groźnej funkcji **i** da się go namówić na jej wywołanie.

- **01 — excessive agency**: chatbot ma podpięty `debug_sql` (surowy SQL) → `DELETE FROM users WHERE username='carlos'`. Metoda: enumeruj funkcje (pytaj model wprost) → znajdź groźną → podawaj DOKŁADNY SQL (nie każ improwizować).
- **Następne w topicu:** indirect prompt injection.

Powtarzalny schemat: **enumerate funkcje → zidentyfikuj groźną → wywołaj z precyzyjnym payloadem.**

### `path-traversal/` — Path traversal (część „Server-side vulnerabilities")
Aplikacja czyta plik po nazwie z inputu (`read(baza + filename)`). Wsadzasz `../`, wychodzisz poza katalog bazowy i czytasz dowolny plik. **Prymityw READ, nie execute** — nie ma `ls`/komend (to OS command injection). Ta sama mechanika `..` co w normalizacji cache deception.

- **01 — simple case**: `GET /image?filename=../../../../../etc/passwd` (brak zabezpieczeń, goły `../`).
- **Następne:** obejścia filtrów (absolutna ścieżka, `....//`, URL-encode `%2e%2e%2f` / podwójny, wymuszony prefiks, null byte `%00`, wymuszone rozszerzenie).

### `access-control/` — Access control (część „Server-side vulnerabilities")
Trzecia warstwa po authentication („kim jesteś") i session („czy to nadal ty"): **authorization** = „czy **wolno ci** to zrobić". Broken access control = ta warstwa zawodzi mimo poprawnego logowania. OWASP Top 10 #1.

- **01 — unprotected admin functionality**: panel admina bez żadnego checka uprawnień; ścieżka (losowa) wyczytana z `robots.txt` → wejście na `/administrator-panel` → `Delete` carlosa (GET). Sedno: `robots.txt` to mapa skarbów, nie ochrona (security through obscurity); **strona ≠ akcja** (każdy endpoint musi mieć własny check).

Powtarzalny schemat: **znajdź wrażliwą funkcję → sprawdź czy endpoint realnie weryfikuje uprawnienia → wywołaj bezpośrednio (URL/parametr)**.

> **Uwaga:** przeszliśmy z Web LLM attacks (⏸ pauza po labce 01) na foundacyjną ścieżkę **Server-side vulnerabilities**, bo dalsze labki LLM opierają się na klasycznych podatnościach (command injection, XSS…). Kolejność: fundamenty server-side → client-side → powrót do LLM.
