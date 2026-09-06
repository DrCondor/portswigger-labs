# Lab 04 — Exploiting cache server normalization for web cache deception

- **Kategoria:** Web cache deception → normalization discrepancies → **cache server normalization** (+ delimiter discrepancy)
- **Poziom:** Practitioner
- **Data:** 2026-09-06
- **Status:** ✅ rozwiązany
- **Narzędzia:** Burp Suite Community (Proxy + **Intruder** + Repeater + **Inspector**), exploit server

---

## Cel labu

Wykraść **API key użytkownika carlos**. Tym razem rozbieżność jest **złożeniem dwóch trików**: delimiter (Lab 02) + normalizacja (Lab 03, ale odwrócona).

---

## TL;DR — dwa zdania

Origin **nie normalizuje** i **ucina ścieżkę na `#`** (delimiter) → widzi `/my-account`. Cache **normalizuje** (dekoduje `%2f`, rozwija `..`, a `#` traktuje jak zwykły znak) → widzi `/resources/<plik>`. Payload: **`/my-account%23%2f..%2fresources%2fwcd`**.

---

## Czym różni się od Lab 03 (lustro)

| | Lab 03 (origin normalizuje) | **Lab 04 (cache normalizuje)** |
|---|---|---|
| Kto normalizuje | origin | **cache** |
| Kto bierze dosłownie | cache | **origin (+ ucina na delimiterze)** |
| Gdzie `/resources` w payloadzie | **z przodu** | **z tyłu** |
| Gdzie `/my-account` | z tyłu | **z przodu** |
| Potrzebny delimiter? | nie (origin sam zwija traversal) | **TAK** (origin nie zwija, więc trzeba go uciąć) |
| Payload | `/resources/..%2fmy-account` | `/my-account%23%2f..%2fresources%2fwcd` |

Statyczny „znacznik" i prywatny endpoint **zamieniają się miejscami**.

---

## Dlaczego tu potrzeba delimitera (klucz do zrozumienia)

W Lab 03 origin **normalizował**, więc `/resources/..%2fmy-account` sam się zwijał do `/my-account`.

Tutaj origin **NIE normalizuje** — bierze ścieżkę dosłownie. Gdybyś dał `/my-account%2f..%2fresources%2fwcd`, origin zobaczyłby cały ten literalny potworek → szukałby route'a → **404**. Origin odda konto **tylko gdy ścieżka to dokładnie `/my-account`**. Skoro nie zwija traversala, jedyny sposób to **uciąć śmieć delimiterem**. Stąd cała faza szukania delimitera (jak Lab 02).

---

## Znalezienie delimitera (Burp Intruder)

Setup jak Lab 02: pozycja tuż za `/my-account` → `GET /my-account§§ HTTP/2`, Sniper, lista delimiterów, **URL-encoding wyłączony**.

Czytanie wyników (baza `/my-account` = 200, Length 4078):

| payload | status | length | werdykt |
|---|---|---|---|
| `#` / `%23` | 200 | **4078** (pełne konto) | ✅ **delimiter** |
| `?` / `%3F` | 200 | 4078 | ❌ query string (fałszywy pozytyw jak w Lab 02) |
| `/` / `%2F` | 200 | 4012 (inna strona) | ❌ trailing slash, nie to |
| reszta | 404 | 131 | origin nie zna znaku |

**Dlaczego `?` odpada, choć wygląda tak samo jak `#`:** przy `?` cache też widzi query string → jego ścieżką jest gołe `/my-account` (nie statyk) → **nie zacachuje**. Query zabija sztuczkę po stronie cache'a. Przy `#` cache trzyma znak w ścieżce → `..` może się przewinąć do `/resources`.

---

## ⭐⭐ Najważniejszy myk: `#` MUSISZ wysłać jako `%23`

`#` w **przeglądarce** zaczyna *fragment* — wszystko po nim **nie jest wysyłane na serwer**. Skoro prawdziwy atak leci przez przeglądarkę carlosa, gołe `#` w URL-u = przeglądarka utnie do `/my-account` → cache nie zobaczy `/resources` → **atak pada po cichu**.

Dlatego **`%23`**: przeglądarka nie rozpoznaje fragmentu, wysyła znak surowo jako część ścieżki. Dopiero:
- **origin** dekoduje `%23`→`#` i **ucina** (delimiter) → `/my-account`
- **cache** dekoduje `%23`→`#`, ale traktuje jak **zwykły znak** (nie fragment!) i rozwija resztę → `/resources/wcd`

To, że *ten sam znak* jest delimiterem origina i fragmentem przeglądarki, to sedno labu. Gołe `#` zawsze zawiedzie; ratuje enkodowanie.

**Gdzie czego używasz:**

| Gdzie | Znak | Po co |
|---|---|---|
| Intruder (recon) | `#` i `%23` | szukanie delimitera — dlatego oba dają 200 |
| Exploit (przez przeglądarkę carlosa) | **`%23`** | gołe `#` przeglądarka utnie |
| Repeater (odczyt) | `%23` | ten sam string → ten sam klucz w cache |

---

## Finalny payload

```
/my-account%23%2f..%2fresources%2fwcd
```

**Origin** (dekoduje `%23`→`#`, ucina): `/my-account` → konto carlosa ✅
**Cache** (normalizuje `%23`→`#` jako znak, `%2f`→`/`, rozwija `..`):
```
/my-account#/../resources/wcd
  "my-account#" ← skasowane przez ..
→ /resources/wcd → static directory → zapisuję ✅
```

Cache trzyma odpowiedź origina (konto carlosa) pod kluczem, który po normalizacji wygląda na statyczny.

---

## Metodologia rekonesansu (kolejność testów)

1. **Origin NIE normalizuje** → `GET /aaa%2f..%2fmy-account` = **404** (w Lab 03 było 200). Odwrócony kierunek potwierdzony.
2. **Cache normalizuje** → `GET /aaa%2f..%2fresources/js/tracking.js`: origin by to znormalizował na 404, ale **cache oddaje `X-Cache: hit` z treścią tracking.js** — bo znormalizował ścieżkę do `/resources/js/tracking.js` i podał z cache'a, zanim dotknął origina. ✅
3. **Delimiter** → Intruder (wyżej).
4. **Test płytko na sobie** → payload ze **świeżą nazwą** 2× → `miss`→`hit` = mechanizm działa.

---

## 🔥 Debugowanie dostarczenia — tu spędziliśmy 80% czasu

Payload był poprawny szybko. Diabeł siedział w dostarczeniu. Lista wpadek (wszystkie dają ten sam objaw: `X-Cache: miss` w nieskończoność):

### 1. Podwójny slash (`%2f/` → `//`)
`...%2f..%2f/resources` po dekodowaniu = `..//resources`. **Inspector → „Decoded from URL encoding"** pokazał to od ręki. → Nie mieszaj `%2f` z gołym `/`.

### 2. Ścieżka kończąca się na gołym katalogu
`.../resources` (sam katalog) **nie łapie się** na regułę static directory — ta chce **plik POD katalogiem**. Musi być `/resources/<coś>`. To był powód uporczywego `miss` mimo poprawnej reszty.

### 3. Kropki — kiedy kodować, kiedy nie
`..` kodujesz (`%2e%2e`) **tylko gdy stoi między prawdziwymi `/`** (`/../`) — wtedy przeglądarka zwija je sama. U nas `..` siedzi między `%2f` (`%2f..%2f`) → nic go po drodze nie rusza → **zwykłe `..` wystarcza**. Reguła: *enkodujesz to, co inaczej zadziałałoby za wcześnie — slashe zawsze, kropki tylko między prawdziwymi slashami.*

### 4. Ściganie się z ofiarą (self-poisoning) ⭐
Przy `cache server normalization` **każdy Twój odczyt to broń obosieczna**: jak trafisz `miss`, Twój request **od razu cachuje TWOJE konto** pod tą nazwą. Jak sprawdzisz „od razu" po Deliver (zanim bot wejdzie), zatruwasz wpis swoim kontem → carlos dostaje Twoją kopię → jego dane nigdy nie lądują.
**Fix:** carlos **pierwszy**, Ty **drugi**. Potwierdź wejście bota w **Access log** (`GET /exploit/` od `(Victim)`, inne IP) — **bez dotykania cache'a** — i dopiero potem strzel **raz** w Repeaterze. Podejrzewasz zatrucie → świeża nazwa + cierpliwość (wpis i tak wygasa po 30 s).

### 5. Brak `Store` przed `Deliver`
Bez `Store` exploit server oddaje botowi **pustą/starą** stronę → carlos wchodzi na `/exploit/`, ale bez skryptu → brak redirectu. W logu widać wejście `(Victim)`, więc wygląda OK — a nic się nie dzieje. **Zawsze: Store → Deliver.**

### 6. Niedomknięty cudzysłów w skrypcie ⭐ (najbardziej podstępny)
```html
<script>document.location="https://.../resources%2fbbb</script>   ← brak zamykającego "
```
String w JS niedomknięty → `SyntaxError` → skrypt **w ogóle się nie wykonuje** → brak przekierowania. Objaw identyczny jak wyżej: bot wchodzi na `/exploit/`, cache pusty. **Cichy błąd — patrzyliśmy na cache, a wina była w składni JS-a.**
**Lekcja:** jak exploit „nic nie robi", **otwórz go sam w Chrome / Burp Browser i zerknij w konsolę** — literówka w skrypcie wywali się głośno.

---

## Poprawna procedura exploita (checklist)

1. Exploit body ze **świeżą nazwą** i **domkniętym `"`**:
   ```html
   <script>document.location="https://LAB-ID.web-security-academy.net/my-account%23%2f..%2fresources%2fsteal1"</script>
   ```
2. **Store** (obowiązkowo!).
3. **Deliver exploit to victim.**
4. **Access log** → poczekaj na `(Victim)` na `/exploit/` (nie dotykaj cache'a).
5. **Raz** w Repeaterze: `GET /my-account%23%2f..%2fresources%2fsteal1` → `X-Cache: hit` + w body **klucz carlosa** → **Submit**.

---

## Pojęcia do zapamiętania

- **cache server normalization** — cache normalizuje, origin nie (lustro Lab 03)
- **złożenie rozbieżności** — delimiter (Lab 02) + normalizacja (Lab 03) w jednym payloadzie
- **`#` jako delimiter + `%23`** — ten sam znak: delimiter origina i fragment przeglądarki → musi być zakodowany
- **Burp Inspector „Decoded from URL encoding"** — pokazuje, co naprawdę wysyłasz; łapie `//`, złe traversale
- **self-poisoning przy odczycie** — Twój `miss` cachuje Twoje konto; ofiara musi być pierwsza
- **Access log jako sonda** — potwierdza wejście bota bez dotykania cache'a
- **cichy `SyntaxError` w exploicie** — brak `"`/złe `</script>` = skrypt nie działa, objaw jak „cache nie łapie"

---

## Mindset (co się powtarza, co nowe)

Schemat **znajdź rozbieżność → dostarcz ofierze → czytaj z cache** — ten sam.

**Nowe 1:** rozbieżności można **składać** (delimiter + normalizacja). Kierunek (kto normalizuje) decyduje, po której stronie payloadu jest statyk.

**Nowe 2— najważniejsze z tego labu:** **poprawny payload to połowa roboty.** Druga połowa to *dostarczenie*: Store, timing (ofiara pierwsza), poprawna składnia exploita. Gdy „nie działa" mimo dobrego payloadu — **zawężaj: czy bot wszedł (Access log)? czy skrypt się wykonał (konsola w przeglądarce)? czy nie zatrułem sam (świeża nazwa)?** To jest systematyczne debugowanie, nie zgadywanka.

**Przenośne pytanie do każdej klasy podatności:** *czy komponent dekoduje/rozwija, czy bierze dosłownie — i czy któryś znak ma dla niego specjalne znaczenie (delimiter/fragment/query)?*

Cały temat **Web cache deception** domknięty (labki 01–04): path mapping, delimiters, origin normalization, cache normalization.
