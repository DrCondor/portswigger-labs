# Lab 02 — Exploiting path delimiters for web cache deception

- **Kategoria:** Web cache deception → static extension cache rules → delimiter discrepancies
- **Poziom:** Practitioner
- **Data:** 2026-08-23
- **Status:** ✅ rozwiązany
- **Narzędzia:** Burp Suite Community (Proxy + **Intruder** + Repeater), exploit server

---

## Cel labu

Wykraść **API key użytkownika carlos** (jak w Lab 01), tym razem przez **rozbieżność delimiterów**.

---

## Czym różni się od Lab 01

| | Lab 01 (path mapping) | Lab 02 (delimiters) |
|---|---|---|
| Jak origin ucina ścieżkę | ignoruje cały sufiks po `/` (`/my-account/foo`) | ucina na **znaku-delimiterze** (`/my-account;foo`) |
| Test `/my-account/aaa` | wraca konto (200) | **404** — path mapping nie działa |
| Trik | doklej `/x.js` | znajdź delimiter, potem doklej `;x.js` |
| Reguła cache | static extension `.js` | static extension `.js` (ta sama) |

**Wniosek:** jak `/my-account/aaa` daje 404, path mapping odpada → sprawdzaj delimitery.

---

## Koncept: rozbieżność delimiterów

Delimiter = znak wyznaczający granicę w URL-u. Standardy są luźne (RFC pozwala na sporo), więc różne frameworki traktują różne znaki jako delimiter.

- Origin traktuje jakiś znak jako delimiter → **ucina ścieżkę** w tym miejscu → oddaje dynamiczną stronę.
- Cache nie zna tego znaku → widzi **całą ścieżkę z `.js`** → cache'uje.

Przykłady frameworków:
- **Java Spring** → `;` (matrix variables) ← **to był delimiter w tym labie**
- **Ruby on Rails** → `.` (format odpowiedzi, np. `.json`)
- **OpenLiteSpeed** → zakodowany null `%00`

---

## Mechanizm tego labu

Payload: `/my-account;aaa.js`
- **origin** (Spring, `;` = delimiter): ucina na `;` → czyta `/my-account` → oddaje konto z API key
- **cache** (nie zna `;`): widzi całość `/my-account;aaa.js` → kończy się `.js` → zapisuje

---

## Metodologia rekonesansu (2 kroki)

### Krok 1 — znajdź delimiter origina (Burp Intruder)
Potrzebujesz 3 punktów odniesienia:
1. **Baza:** `GET /my-account` → konto + API key (200).
2. **Referencja:** `GET /my-accountaaa` → jak wygląda „porażka" (tu: 404). *Gdyby == baza → redirect, zmień endpoint.*
3. **Test:** `GET /my-account<ZNAK>aaa` dla każdego znaku z listy.
   - odpowiedź == baza (konto) → **znak jest delimiterem** ✅
   - odpowiedź == referencja (404) → nie jest

**Setup Intrudera:**
- Wyślij request do Intrudera, w linii 1 zrób `GET /my-account§§aaa HTTP/2` (pusta pozycja `§§` między `/my-account` a `aaa`).
- Tryb **Sniper**.
- W **Payloads** wklej listę delimiterów (jeden znak na linię).
- ⚠️ **WYŁĄCZ URL-encoding** (Payloads → Payload encoding → odznacz „URL-encode these characters"). Inaczej `;` → `%3b` i test jest bezużyteczny.
- **Start attack.** (Community throttluje Intrudera — wolniej, ale dla ~30 znaków OK.)

**Czytanie wyników:** kolumna **Length**. Większość znaków = jedna powtarzalna długość (404). Outlier z inną długością + status 200 = kandydat na delimiter.
- W tym labie: prawie wszystko `158` (404), a `;` i `?` dały `200` / `4072` (pełne konto).

### ⚠️ Pułapka: `?`
`?` zawsze zwróci stronę konta, bo zaczyna query string — to NIE jest rozbieżność, tylko normalne zachowanie. Fałszywy pozytyw. **Odrzuć `?`.** Prawdziwy delimiter to `;`.

### Krok 2 — potwierdź regułę cache (Repeater)
`GET /my-account;aaa.js` → wyślij 2×, patrz na `X-Cache: miss` → `hit`. ✅
Jak `.js` nie łapie: próbuj `.css`, `.ico`, `.exe`.

**Tip:** przy testach dodawaj zmienny cache buster (`?cb=1`, `?cb=2`...), żeby cache nie podawał starych odpowiedzi i nie fałszował wyników.

---

## Payload (finalny)

```
/my-account;wcd1.js
```

## Faza exploita — IDENTYCZNA jak Lab 01

Skrypt na exploit serverze (świeża nazwa pliku!):
```html
<script>document.location="https://LAB-ID.web-security-academy.net/my-account;wcd1.js"</script>
```
Potem: **Store → Deliver to victim → (Access log) → szybko Repeater** `GET /my-account;wcd1.js` → `X-Cache: hit` + klucz carlosa → Submit solution.

Pełny diagram przepływu ataku → patrz `01-path-mapping.md`. Zmienił się tylko payload (delimiter zamiast path mappingu), reszta bez zmian.

---

## Pułapki (te same co Lab 01 + nowe)

- **Cache żyje ~30 s** — Deliver → od razu Repeater.
- **Nie odpytuj payloadu sam przed carlosem** — zacachujesz swoje konto. Jak się stanie: świeża nazwa (`wcd2.js`).
- **Store przed Deliver.**
- **[NOWE] Wyłącz encoding w Intruderze** — inaczej delimitery się zakodują.
- **[NOWE] `?` to fałszywy pozytyw** — pomijaj przy analizie wyników.

---

## Lista delimiterów od PortSwiggera (do reużycia)

Surowe znaki + wersje zakodowane. Do Intrudera wklejać **jeden na linię** i **z wyłączonym URL-encoding**.

```
! " # $ % & ' ( ) * + , - . / : ; < = > ? @ [ \ ] ^ _ ` { | } ~
%21 %22 %23 %24 %25 %26 %27 %28 %29 %2A %2B %2C %2D %2E %2F %3A %3B %3C %3D %3E %3F %40 %5B %5C %5D %5E %5F %60 %7B %7C %7D %7E
```

Źródło: https://portswigger.net/web-security/web-cache-deception/wcd-lab-delimiter-list

**Uwaga o wersjach zakodowanych:** przydają się przy *delimiter decoding discrepancies* — gdy origin/cache dekoduje znak przed przetworzeniem (np. `%23` → `#`). Niektóre znaki (`{`, `}`, `<`, `>`, `#`) przeglądarka i tak zakoduje/utnie sama, więc w realnym exploicie trzeba wersji zakodowanej.

---

## Pojęcia do zapamiętania

- **delimiter discrepancy** — origin i cache różnie interpretują znak jako granicę ścieżki
- **matrix variables** (Spring, `;`) — stąd `;` jako delimiter
- **static extension cache rule** — cache zapisuje po rozszerzeniu (`.js`, `.css`, `.ico`)
- **Burp Intruder / Sniper** — automatyczne podstawianie payloadów w jedną pozycję
- **Length jako sygnał** — outlier w długości odpowiedzi = coś się różni = trop
- **cache buster** — zmienny query param, żeby wymusić świeżą odpowiedź podczas testów

---

## Mindset (co się powtarza)

Schemat **znajdź rozbieżność → dostarcz ofierze → czytaj z cache** jest ten sam co w Lab 01.
Nowość: **rozbieżność może być w wielu miejscach** (mapowanie ścieżki, delimitery, normalizacja) — a metoda szukania jest zawsze podobna: porównuj odpowiedzi (baza vs referencja vs test) i patrz, gdzie origin i cache się „rozjeżdżają".

Następne w temacie: **normalization discrepancies** (static directory rules + path traversal `..%2f`).