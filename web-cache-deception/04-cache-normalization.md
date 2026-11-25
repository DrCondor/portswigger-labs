# Lab 04 — Exploiting cache server normalization for web cache deception

| | |
|---|---|
| **Poziom** | Practitioner |
| **Status** | ✅ rozwiązany |
| **Narzędzia** | Burp Community (Proxy + **Intruder** + Repeater + **Inspector**), exploit server |
| **Data** | 2026-09-06 |

> 📖 **Teoria tej klasy podatności: [README tematu](README.md).** Tu tylko ten konkretny lab.

---

## Cel

Wykraść **API key carlosa**. Rozbieżność to **złożenie dwóch trików**: delimiter ([[02-path-delimiters]]) + normalizacja ([[03-origin-normalization]], ale odwrócona).

---

## Trik tego labu (lustro labu 03)

Origin **nie normalizuje** i **ucina ścieżkę na `#`** (delimiter) → widzi `/my-account`. Cache **normalizuje** (dekoduje `%2f`, rozwija `..`, a `#` traktuje jak zwykły znak) → widzi `/resources/<plik>`. Statyczny znacznik i prywatny endpoint **zamieniają się miejscami** względem labu 03.

Payload: **`/my-account%23%2f..%2fresources%2fwcd`**
- **origin** (dekoduje `%23`→`#`, ucina na delimiterze): `/my-account` → konto carlosa,
- **cache** (normalizuje): `/my-account#/../resources/wcd` → `my-account#` skasowane przez `..` → `/resources/wcd` → static directory → zapisuję.

### Dlaczego tu potrzeba delimitera

W labie 03 origin normalizował, więc traversal sam się zwijał. Tu origin **bierze ścieżkę dosłownie** — odda konto tylko gdy ścieżka to dokładnie `/my-account`. Skoro nie zwija traversala, jedyny sposób to **uciąć śmieć delimiterem** (`#`). Stąd faza szukania delimitera jak w labie 02.

### ⭐⭐ Najważniejszy myk: `#` MUSISZ wysłać jako `%23`

`#` w **przeglądarce** zaczyna *fragment* — wszystko po nim **nie jest wysyłane na serwer**. Skoro atak leci przez przeglądarkę carlosa, gołe `#` = przeglądarka utnie do `/my-account`, cache nie zobaczy `/resources` → **atak pada po cichu**. Dlatego `%23`: przeglądarka nie rozpoznaje fragmentu, wysyła znak surowo; dopiero origin dekoduje i ucina, a cache dekoduje i traktuje jak zwykły znak. Ten sam znak jest delimiterem origina i fragmentem przeglądarki — to sedno labu.

| Gdzie | Znak | Po co |
|---|---|---|
| Intruder (recon) | `#` i `%23` | szukanie delimitera (oba dają 200) |
| Exploit (przez przeglądarkę) | **`%23`** | gołe `#` przeglądarka utnie |
| Repeater (odczyt) | `%23` | ten sam string → ten sam klucz w cache |

Delimiter znaleziony Intruderem (baza `/my-account` = 200/4078): `#`/`%23` → 200/4078 ✅; `?` → query string (fałszywy pozytyw — cache nie zacachuje); `/` → inna strona; reszta → 404.

---

## Przebieg

Rekon: (1) origin NIE normalizuje → `GET /aaa%2f..%2fmy-account` = **404** (w labie 03 było 200); (2) cache normalizuje → `GET /aaa%2f..%2fresources/js/tracking.js` → `X-Cache: hit` z treścią tracking.js; (3) delimiter przez Intruder; (4) test na sobie ze świeżą nazwą 2× → `miss`→`hit`. Delivery jak w [README](README.md), payload:
```
/my-account%23%2f..%2fresources%2fsteal1
```

---

## 💥 Moje potknięcia — tu spędziłem 80% czasu

Payload był poprawny szybko. Diabeł siedział w **dostarczeniu**. Wszystkie te wpadki dają ten sam objaw: `X-Cache: miss` w nieskończoność.

1. **Podwójny slash (`%2f/` → `//`)** — `...%2f..%2f/resources` po dekodowaniu = `..//resources`. **Burp Inspector → „Decoded from URL encoding"** pokazał to od ręki. Nie mieszaj `%2f` z gołym `/`.
2. **Ścieżka kończąca się na gołym katalogu** — `.../resources` (sam katalog) **nie łapie się** na static directory; reguła chce **plik POD katalogiem** (`/resources/<coś>`). Powód uporczywego `miss` mimo poprawnej reszty.
3. **Kropki — kiedy kodować** — `..` kodujesz (`%2e%2e`) tylko gdy stoi między prawdziwymi `/` (`/../`, bo wtedy przeglądarka zwija je sama). U nas `..` siedzi między `%2f` → nic go nie rusza → zwykłe `..` wystarcza. Reguła: *enkodujesz to, co inaczej zadziała za wcześnie — slashe zawsze, kropki tylko między prawdziwymi slashami.*
4. **⭐ Self-poisoning przy odczycie** — przy cache normalization każdy Twój `miss` **od razu cachuje TWOJE konto** pod tą nazwą. Sprawdzisz „od razu" po Deliver (zanim bot wejdzie) → zatruwasz wpis swoim kontem → carlos dostaje Twoją kopię. **Fix: carlos pierwszy, Ty drugi.** Potwierdź wejście bota w **Access log** (`(Victim)`, inne IP) **bez dotykania cache'a**, dopiero potem strzel **raz**.
5. **Brak `Store` przed `Deliver`** — exploit server oddaje botowi pustą/starą stronę → brak redirectu. W logu widać wejście `(Victim)`, więc wygląda OK, a nic się nie dzieje.
6. **⭐ Niedomknięty cudzysłów w skrypcie** (najpodstępniejszy) — `document.location="https://.../resources%2fbbb</script>` (brak zamykającego `"`) → `SyntaxError` → skrypt **w ogóle się nie wykonuje**. Objaw identyczny: bot wchodzi, cache pusty. Patrzyłem na cache, a wina była w składni JS. **Lekcja: jak exploit „nic nie robi", otwórz go sam w przeglądarce i zerknij w konsolę** — literówka wywali się głośno.

**Checklist exploita:** świeża nazwa + domknięty `"` → **Store** → **Deliver** → Access log czeka na `(Victim)` → **raz** Repeater → `hit` + klucz carlosa → Submit.

---

## Czego ten lab dokłada

Domknięcie tematu: rozbieżności można **składać** (delimiter + normalizacja), a kierunek (kto normalizuje) decyduje, po której stronie payloadu jest statyk. Najważniejsza przenośna lekcja całego tematu: **poprawny payload to połowa roboty — druga połowa to dostarczenie**, a gdy „nie działa" mimo dobrego payloadu, debuguj systematycznie (bot wszedł? skrypt się wykonał? nie zatrułem sam?), zamiast zgadywać. Temat **Web cache deception domknięty (01–04)**.
