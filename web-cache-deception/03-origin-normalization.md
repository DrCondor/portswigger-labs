# Lab 03 — Exploiting origin server normalization for web cache deception

- **Kategoria:** Web cache deception → normalization discrepancies → **origin server normalization**
- **Poziom:** Practitioner
- **Data:** 2026-09-05
- **Status:** ✅ rozwiązany
- **Narzędzia:** Burp Suite Community (Proxy + Repeater), exploit server

---

## Cel labu

Wykraść **API key użytkownika carlos** (jak w Lab 01/02), tym razem przez **rozbieżność w normalizacji ścieżki** (path traversal z zakodowanym slashem).

---

## Czym różni się od poprzednich labów

| | Lab 01 (path mapping) | Lab 02 (delimiters) | **Lab 03 (normalization)** |
|---|---|---|---|
| Jak origin „widzi" nasz URL | ignoruje sufiks po `/` | ucina na delimiterze (`;`) | **rozwija traversal** `..%2f` |
| Reguła cache | static extension `.js` | static extension `.js` | **static directory** `/resources` |
| Payload | `/my-account/wcd.js` | `/my-account;wcd.js` | `/resources/..%2fmy-account` |
| Kto „ma rację" wg RFC | — | — | **cache** (origin jest „rozlazły", dekoduje `%2f`) |

**Nowość:** rozbieżność nie jest w sufiksie ani w delimiterze, tylko w tym **jak bardzo każdy komponent normalizuje ścieżkę** (dekoduje `%2f`, rozwija `..`).

---

## Topologia (żeby dobrze rozumieć „kto co robi")

```
[Przeglądarka] ──req──> [Cache / CDN] ──req──> [Origin server]
                          (reverse proxy         (aplikacja)
                           z przodu)
```

Cache to **proxy stojące przed** originem — każdy request fizycznie przez nie przechodzi. Ale **to NIE jest tak, że request „idzie przez folder /resources".** To **dwie maszyny czytają ten sam string adresu i decydują niezależnie:**

- **Cache:** „czy mam **zapisać** odpowiedź na ten adres?" → patrzy na *dosłowny* string.
- **Origin:** „jaką **treść** zwrócić?" → patrzy na string *po normalizacji*.

Jeden adres, dwóch czytelników, dwie różne decyzje → cache zapisuje prywatną odpowiedź pod kluczem, który *wygląda* na statyczny.

---

## Koncept: normalization discrepancy

**Normalizacja** = sprowadzenie ścieżki do postaci kanonicznej. Dwie operacje:
1. **dekodowanie procentowe** — `%2f` → `/` (`%2f` = zakodowany ukośnik)
2. **rozwijanie dot-segmentów** — `/foo/bar/../baz` → `/foo/baz` (`..` = katalog wyżej)

Ścieżka URL ma hierarchię jak katalogi w systemie plików (ale to route'y, nie pliki!). `..` cofa o poziom:
```
/resources/../my-account   →   /my-account
   │         │      │
 wejdź    cofnij  wejdź
```

Sedno: **origin i cache normalizują w różnym stopniu.** Chodzi o to, czy dany komponent „przejrzy przez" `..%2f`:
- **dekoduje `%2f`→`/` i rozwija `..`** → widzi ścieżkę po traversalu
- **traktuje `%2f` jako zwykły znak w nazwie segmentu** → traversal go nie rusza, widzi jeden nierozdzielny segment

---

## ⭐ Klucz: dlaczego slash MUSI być zakodowany (`%2f`)

To jest cała istota triku. `/` to **separator segmentów**. Parser tnie ścieżkę po `/`, a segment `..` **kasuje poprzedni**. `%2f` to dosłowny ukośnik będący *częścią nazwy segmentu* — dopóki ktoś go nie zdekoduje.

Ten sam string `/resources/..%2fmy-account`, dwa cięcia:

**Cache (NIE dekoduje `%2f` przed cięciem):**
```
/resources/..%2fmy-account
 └─ "resources"
 └─ "..%2fmy-account"   ← JEDEN segment (%2f to dla niego zwykły znak)
```
Brak samodzielnego `..` → **nic do rozwinięcia** → widzi `/resources/<coś>` → **static directory → cachuję**.

**Origin (NAJPIERW dekoduje `%2f`→`/`, potem tnie):**
```
%2f → /  daje  /resources/../my-account
 └─ "resources"
 └─ ".."          ← kasuje "resources"
 └─ "my-account"
                 → /my-account → prywatna strona z API key
```

**Dlaczego nie zwykły `/` (`/resources/../my-account`)?** Dwa powody:
1. **Przeglądarka sama rozwinie `..` ZANIM wyśle** — na serwer poleci gołe `/my-account`, cache nigdy nie zobaczy `/resources`. Kodując slash, chowasz granicę `..` też przed przeglądarką → forwarduje surowo.
2. **Zniknęłaby rozbieżność** — gdyby oba widziały zwykłe `/`, rozwinęłyby identycznie. `%2f` to **klin, który rozjeżdża interpretacje** (naiwny cache vs dekodujący origin). Enkodowanie *jest* źródłem podatności.

---

## Mechanizm tego labu: origin normalizuje, cache nie

Payload: **`/resources/..%2fmy-account`**
- **origin** (normalizuje): dekoduje `%2f`, rozwija `..` → `/my-account` → konto carlosa z API key
- **cache** (nie normalizuje): widzi `/resources/...` → reguła **static directory** → zapisuje

---

## Procedura rekonesansu (krok po kroku)

### Krok 1 — czy ORIGIN normalizuje?
Weź `/my-account` do Repeatera i **doklej z przodu** śmieciowy segment + traversal:
```
GET /aaa%2f..%2fmy-account HTTP/2
```
- `aaa` = cokolwiek, jest tam **tylko po to, żeby `..` miał co skasować**.
- To **czysty probe**: `/aaa%2f..%2fmy-account` to z pozoru bezsensowna ścieżka. Jedyny sposób, żeby *mimo to* wróciło konto (200 + API key), to taki, że origin zdekodował `%2f`, rozwinął `..` i wylądował na `/my-account`.
- **200 + widać API key** → origin normalizuje ✅
- **404 / login** → origin NIE normalizuje (to byłby wariant *cache* normalization — patrz „następne w temacie")

*W tym labie: 200, strona konta.* ✅

### Krok 2 — czy cache zapisuje jakiś statyczny katalog?
Znajdź w odpowiedzi (albo w Proxy history) zasób ładowany z `/resources/...` — tu widać je wprost w HTML-u strony konta:
```
/resources/js/tracking.js
/resources/css/labs.css
/resources/labheader/...
```
Wyślij `GET /resources/js/tracking.js` 2× i patrz nagłówki:
- `X-Cache: miss` → `hit`, `Cache-Control: max-age=30`, `Age: N` → **cache trzyma `/resources`** ✅

### ⚠️ Haczyk: reguła KATALOGU vs reguła ROZSZERZENIA
`tracking.js` jest **jednocześnie** pod `/resources` **i** kończy się na `.js`. Z jednego requestu nie wiesz, czy cache złapał go bo:
- **(A) static directory** — „wszystko pod `/resources/`", czy
- **(B) static extension** — „wszystko `.js`".

Nasz payload `/resources/..%2fmy-account` **nie kończy się na `.js`** → **potrzebujemy reguły (A)**. Potwierdzasz to Krokiem 3.

### Krok 3 — dry run finalnego payloadu (potwierdza (A) + całość)
```
GET /resources/..%2fmy-account HTTP/2
```
- Wróciło **TWOJE** konto (Ctrl+F → `API Key`)? → origin rozwinął traversal przez prefiks `/resources`. ✅
- `X-Cache: miss` → `hit`? → to **reguła katalogu (A)**, `.js` niepotrzebne. ✅

⚠️ **Ten request cachuje TWOJE konto** pod `/resources/..%2fmy-account` na ~30 s. Jak potem wyślesz carlosowi *ten sam* URL, dostanie Twoją kopię. Dlatego: **albo w ogóle nie odpalaj payloadu na sobie, albo dla ofiary użyj świeżej ścieżki** (patrz niżej). *(Ja w ogóle nie robiłem dry runu na sobie — od razu świeża ścieżka dla carlosa.)*

---

## ⭐ Trik: świeża ścieżka bez zmiany celu (cache busting przez traversal)

Tu, inaczej niż w Lab 01/02, **nie ma „nazwy pliku" do podmiany** (`wcd1.js`→`wcd2.js`). Ale świeży klucz w cache robisz **dokładając śmieciowy katalog + o jeden `..%2f` więcej**:

```
/resources/wcd/..%2f..%2fmy-account
```
Rozwijanie u origina:
```
/resources/wcd/../../my-account
  wcd  ← kasuje 1. ..
resources ← kasuje 2. ..
          → /my-account ✅
```
Cache widzi świeży string `/resources/wcd/..%2f..%2fmy-account` (zaczyna się od `/resources/`) → **nowy wpis, nie kolidujący z ewentualnym własnym cache'em**. Zmieniając `wcd` → `wcd2` → `wcd3` masz tyle świeżych kluczy, ile chcesz.

**Zasada ogólna:** każdy dodatkowy śmieciowy segment przed traversalem wymaga jednego dodatkowego `..%2f`, żeby wrócić na `/my-account`.

---

## Faza exploita (delivery — jak w Lab 01/02)

Skrypt na exploit serverze (świeża ścieżka!):
```html
<script>document.location="https://LAB-ID.web-security-academy.net/resources/wcd/..%2f..%2fmy-account"</script>
```
Potem: **Store → Deliver to victim → (Access log) → szybko Repeater** na **tę samą** ścieżkę:
```
GET /resources/wcd/..%2f..%2fmy-account HTTP/2
```
→ `X-Cache: hit` + API key carlosa → **Submit solution**.

Pełny diagram przepływu ataku (ofiara → exploit server → redirect z jej cookie → cache → my) → patrz `01-path-mapping.md`. Zmienił się tylko payload; mechanika dostarczenia identyczna. **Dlaczego działa:** to przeglądarka carlosa robi żądanie na tę samą domenę, więc automatycznie niesie jego cookie sesyjne → origin oddaje JEGO konto.

---

## Pułapki (te znane + nowe)

- **Cache żyje ~30 s** (`max-age=30`) — Deliver → od razu Repeater.
- **Nie odpalaj payloadu na sobie** (albo użyj świeżej ścieżki) — inaczej zacachujesz swoje konto i carlos dostanie Twoją kopię.
- **Store przed Deliver.**
- **[NOWE] Slash MUSI być zakodowany (`%2f`)** — zwykły `/` przeglądarka rozwinie sama przed wysłaniem, trik zniknie.
- **[NOWE] Reguła katalogu vs rozszerzenia** — upewnij się, że cache łapie po katalogu (`/resources`), bo payload nie ma `.js`.
- **[NOWE] Sprawdź, że traversal wraca na Twoje konto** (Krok 3), nie na stronę główną/login — inaczej origin nie znormalizował tak, jak myślisz.

---

## Pojęcia do zapamiętania

- **normalization / canonicalization** — sprowadzenie URL-a do postaci kanonicznej (dekodowanie `%xx`, rozwijanie `.`/`..`)
- **path traversal** (`../`, `..%2f`) — „cofnij katalog"; tu użyty do climb-up z `/resources` na `/my-account`
- **encoded slash `%2f`** — klin rozjeżdżający interpretacje; przeżywa normalizację przeglądarki i cache'a, rozwija go dopiero origin
- **origin server normalization** — origin normalizuje, cache nie → payload: statyczny prefiks + traversal
- **static directory cache rule** — cache zapisuje wszystko pod danym katalogiem (`/resources`), niezależnie od rozszerzenia
- **dummy segment + `..%2f`** — technika świeżego klucza cache (odpowiednik `wcd1`/`wcd2` z poprzednich labów)
- **reverse proxy topology** — cache stoi przed originem; oba czytają ten sam URL niezależnie

---

## Mindset (co się powtarza, co nowe)

Schemat **znajdź rozbieżność → dostarcz ofierze → czytaj z cache** — ten sam co Lab 01/02.

**Nowe:** rozbieżność może siedzieć w **stopniu normalizacji ścieżki**, nie tylko w mapowaniu/delimiterach. Metoda szukania wciąż ta sama — **porównuj odpowiedzi (baza vs bezsensowna ścieżka vs test)** i patrz, gdzie origin i cache się rozjeżdżają.

Kolejny wymiar do sprawdzania przy każdej podatnej klasie: **czy komponent dekoduje / rozwija, czy bierze dosłownie?** — to pytanie wraca w SSRF, path traversal, WAF bypassach itd.

**Następne w temacie:** *cache server normalization* — lustrzane odbicie tego labu: **cache normalizuje, origin nie**. Wtedy payload odwraca się (statyczny cel „ukryty" tak, żeby to *cache* rozwinął go do statycznego zasobu, a origin oddał dynamiczną stronę).
