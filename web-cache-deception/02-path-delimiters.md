# Lab 02 — Exploiting path delimiters for web cache deception

| | |
|---|---|
| **Poziom** | Practitioner |
| **Status** | ✅ rozwiązany |
| **Data** | 2026-08-23 |
| **Narzędzia** | Burp Community (Proxy + **Intruder** + Repeater), exploit server |

> 📖 **Teoria tej klasy podatności: [README tematu](README.md).** Tu tylko ten konkretny lab.

---

## Cel

Wykraść **API key carlosa** (jak w [[01-path-mapping]]), tym razem przez **rozbieżność delimiterów**.

---

## Trik tego labu

Path mapping tu **nie działa** (`/my-account/aaa` → 404). Origin (Java Spring) ucina ścieżkę na znaku-delimiterze `;` (matrix variables), a cache tego znaku nie zna.

Payload: **`/my-account;wcd.js`**
- origin (`;` = delimiter): ucina na `;` → czyta `/my-account` → konto z API key,
- cache (nie zna `;`): widzi całość → kończy się `.js` → zapisuje.

> Jak `/my-account/aaa` daje 404 → path mapping odpada → **szukaj delimitera**. Przykłady: Spring `;`, Rails `.`, OpenLiteSpeed `%00`.

### Szukanie delimitera (Intruder Sniper)

Pozycja tuż za `/my-account`: `GET /my-account§§aaa HTTP/2`, Sniper, lista delimiterów (jeden znak/linię), **URL-encoding WYŁĄCZONY** (inaczej `;`→`%3b`). Czytaj kolumnę **Length**: większość = jedna długość (404), outlier 200/pełne konto = kandydat. Tu `;` dał 200/pełne konto.

Potwierdzenie cache (Repeater): `GET /my-account;aaa.js` 2× → `miss`→`hit` ✅.

---

## Przebieg

Rekon: Intruder znajduje `;` → Repeater potwierdza cache na `.js`. Delivery identyczny jak [[01-path-mapping]] (exploit server, Store → Deliver → szybko Repeater), zmienia się tylko payload:
```
/my-account;wcd1.js
```

---

## 💥 Moje potknięcia

- **[NOWE] Nie wyłączony URL-encoding w Intruderze** — delimitery się kodują (`;`→`%3b`) i test jest bezużyteczny. Payloads → Payload encoding → odznacz.
- **[NOWE] `?` to fałszywy pozytyw** — zawsze zwróci konto, bo zaczyna query string; to nie rozbieżność. Odrzuć `?`, prawdziwy delimiter to `;`.
- Te same co lab 01: cache ~30 s (Deliver → od razu Repeater), nie odpytuj payloadu sam przed carlosem (świeża nazwa `wcd2.js`), Store przed Deliver.
- **Cache buster** (`?cb=1`, `?cb=2`…) przy testach, żeby cache nie podawał starych odpowiedzi i nie fałszował wyników.

**Lista delimiterów PortSwiggera** (do Intrudera, encoding OFF): surowe `! " # $ % & ' ( ) * + , - . / : ; < = > ? @ [ \ ] ^ _ \` { | } ~` + wersje `%21…%7E`. Źródło: portswigger.net/web-security/web-cache-deception/wcd-lab-delimiter-list

---

## Czego ten lab dokłada

Druga rozbieżność (**delimiter**) + warsztat szukania go Intruderem. Pokazuje, że rozbieżność może być w wielu miejscach, a metoda szukania jest zawsze podobna (baza vs referencja vs test). Następne: **normalization** ([[03-origin-normalization]]) — rozbieżność w *stopniu* normalizacji ścieżki.
