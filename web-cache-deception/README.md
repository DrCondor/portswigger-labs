# Web cache deception (WCD)

> Cache i origin server **różnie interpretują ten sam URL**. Origin widzi swój prywatny endpoint i oddaje dane konta; cache widzi „statyczny plik" i zapisuje odpowiedź do **wspólnej** pamięci. Podstępem sprawiasz, że przeglądarka ofiary (z jej sesją) wrzuci swoją prywatną stronę do cache'a — potem ty ją stamtąd odczytujesz.

---

## O co chodzi (koncept)

Cache (pamięć podręczna) to warstwa, która zapisuje kopie odpowiedzi serwera, żeby przy kolejnym takim samym żądaniu oddać gotowca zamiast fatygować origin. Zwykle cache'uje się **statyczne pliki** (`.js`, `.css`, obrazki) — bo się nie zmieniają.

Problem: **cache jest wspólny dla wszystkich**. Jeśli podstępem wrzucimy do niego stronę z czyimiś prywatnymi danymi (np. konto z API key), to potem każdy — łącznie z atakującym — może tę kopię wyciągnąć.

Cała klasa stoi na **zderzeniu dwóch interpretacji jednego URL-a**:

| Komponent | Jak czyta URL `/my-account/x.js` |
|---|---|
| **Origin server** | „to strona konta → oddaję prywatne dane zalogowanego usera" |
| **Cache** | „kończy się na `.js` → statyczny plik → zapisuję" |

Ta rozbieżność to właśnie *deception*.

---

## Jak to działa (mechanizm)

### Topologia: cache stoi PRZED originem

```
[Przeglądarka] ──req──> [Cache / CDN] ──req──> [Origin server]
                         (reverse proxy)        (aplikacja)
```

To **dwie maszyny czytają ten sam string adresu i decydują niezależnie:**
- **Cache:** „czy mam **zapisać** odpowiedź na ten adres?" → patrzy na *dosłowny* string.
- **Origin:** „jaką **treść** zwrócić?" → patrzy na string *po swojej normalizacji*.

Jeden adres, dwóch czytelników, dwie decyzje → cache zapisuje prywatną odpowiedź pod kluczem, który *wygląda* na statyczny.

### Dlaczego atak w ogóle działa: cookie ofiary

> To **przeglądarka ofiary** wykonuje żądanie na spreparowany URL — a że to nawigacja na **tę samą domenę**, na której ofiara jest zalogowana, przeglądarka **automatycznie dokleja jej cookie sesyjne**. Origin widzi sesję ofiary → zwraca JEJ dane → cache je zapisuje. Gdyby żądanie szło bez jej cookie, wróciłaby strona logowania i atak by padł.

### Cztery rodzaje rozbieżności (taksonomia tematu)

Rozbieżność może siedzieć w różnych miejscach — stąd 4 laby:

| # | Rodzaj | Na czym polega rozbieżność | Payload (wzór) |
|---|---|---|---|
| 01 | **path mapping** | origin mapuje po prefiksie, **ignoruje sufiks** | `/my-account/wcd.js` |
| 02 | **delimiters** | origin **ucina ścieżkę na znaku-delimiterze** (`;` w Spring), cache nie zna znaku | `/my-account;wcd.js` |
| 03 | **origin normalization** | **origin** dekoduje `%2f`/rozwija `..`, cache bierze dosłownie | `/resources/..%2fmy-account` |
| 04 | **cache normalization** | **cache** normalizuje, origin nie (+ delimiter) | `/my-account%23%2f..%2fresources%2fwcd` |

Pytanie przewodnie przy każdym: **czy komponent dekoduje/rozwija, czy bierze dosłownie — i czy któryś znak ma dla niego specjalne znaczenie (delimiter/fragment/query)?**

---

## Ogólny schemat ataku

**Znajdź rozbieżność → dostarcz link ofierze (exploit server) → czytaj dane z cache.**

Rekonesans rozbieżności (zawsze ten sam szkielet): porównuj odpowiedzi **baza vs referencja/bezsensowna ścieżka vs test** i patrz, gdzie origin i cache się rozjeżdżają. Stan cache czytasz z nagłówków: `X-Cache: miss` → `hit`, `Age`, `Cache-Control: max-age`.

Dostarczenie (identyczne dla wszystkich labów):
1. Skrypt na **exploit serverze** (redirect ofiary na payload): `<script>document.location="https://LAB/<payload>"</script>`
2. **Store** → **Deliver exploit to victim** → (Access log) → **szybko Repeater** na ten sam URL → `X-Cache: hit` + sekret ofiary.

---

## Jak się przed tym bronić

- **Cache'uj tylko to, co naprawdę statyczne** — na podstawie odpowiedzi origina (`Content-Type`, nagłówki `Cache-Control`), nie samego rozszerzenia w URL-u.
- **Spójna normalizacja** — origin i cache powinny interpretować ścieżkę identycznie (ten sam parser / ta sama kanonizacja).
- Origin powinien zwracać `Cache-Control: no-store` dla stron z danymi usera.

---

## Słowniczek

- **web cache deception** — wrzucenie prywatnej strony do wspólnego cache przez rozbieżność interpretacji URL-a.
- **rozbieżność cache vs origin** — sedno całej klasy.
- **path mapping** — origin mapuje po prefiksie, ignoruje sufiks.
- **delimiter discrepancy** — różne znaki-granice (Spring `;`, Rails `.`, OpenLiteSpeed `%00`).
- **normalization / canonicalization** — dekodowanie `%xx`, rozwijanie `.`/`..`.
- **encoded slash `%2f`** — klin rozjeżdżający interpretacje; przeżywa normalizację przeglądarki/cache, rozwija go dopiero ten, kto dekoduje.
- **static extension / static directory rule** — cache zapisuje po rozszerzeniu (`.js`) albo po katalogu (`/resources`).
- **cookie sesyjne przy nawigacji same-origin** — leci automatycznie, dlatego atak działa.
- **X-Cache: miss/hit, Age, Cache-Control: max-age** — nagłówki do czytania stanu cache.
- **exploit server** — serwer atakującego; w realu odpowiednik phishingu (bot = „ofiara kliknęła link").
- **cache buster** — zmienny query param / świeża nazwa, żeby wymusić świeżą odpowiedź podczas testów.

> **Narzędzia:** Burp Community wystarcza (Proxy + Repeater + Intruder do szukania delimiterów + Inspector do podglądu zdekodowanego URL-a). Cache w labach żyje **~30 s** (`max-age=30`) — po Deliver działaj szybko.

---

## Laby w tym temacie

- [01 — path mapping](01-path-mapping.md) — origin ignoruje sufiks → `/my-account/wcd.js`.
- [02 — path delimiters](02-path-delimiters.md) — delimiter `;` (Spring); szukanie Intruderem.
- [03 — origin server normalization](03-origin-normalization.md) — `/resources/..%2fmy-account` (encoded slash, static directory).
- [04 — cache server normalization](04-cache-normalization.md) — `/my-account%23%2f..%2fresources%2fwcd` (`#` jako delimiter wysyłany jako `%23`); dużo debugowania dostarczenia.

---

## Mindset / co się przenosi

Schemat **znajdź rozbieżność → dostarcz ofierze → czytaj z cache** jest stały; zmienia się tylko, **gdzie** siedzi rozbieżność i **po której stronie** payloadu jest statyk. Dwie przenośne lekcje:
1. **Poprawny payload to połowa roboty** — druga połowa to dostarczenie (Store, timing „ofiara pierwsza", poprawna składnia exploita). Gdy „nie działa" mimo dobrego payloadu — debuguj systematycznie: czy bot wszedł (Access log)? czy skrypt się wykonał (konsola)? czy nie zatrułem sam cache'a?
2. Pytanie **„dekoduje/rozwija czy bierze dosłownie?"** wraca wszędzie: path traversal, SSRF, WAF bypassy. To mikro-wersja jednego etapu BSCP — skala inna, schemat myślenia ten sam.
