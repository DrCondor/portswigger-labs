# Lab 01 — File path traversal, simple case

| | |
|---|---|
| **Poziom** | Apprentice |
| **Status** | ✅ rozwiązany |
| **Data** | 2026-09-17 |
| **Narzędzia** | Burp Community (Proxy + Repeater) |

> 📖 **Teoria tej klasy podatności: [README tematu](README.md).** Tu tylko ten konkretny lab.

---

## Cel

Path traversal w wyświetlaniu obrazków produktów. Odczytać zawartość `/etc/passwd`.

---

## Trik tego labu

„Simple case" = **zero zabezpieczeń**, goły `../` przechodzi bez enkodowania. Request ładujący obrazek to `GET /image?filename=53.jpg`; podmieniasz nazwę pliku na traversal do korzenia i dalej do `/etc/passwd`:

```
GET /image?filename=../../../../../etc/passwd HTTP/2
```

→ Response zawiera `root:x:0:0:root:/root:/bin/bash ...` zamiast obrazka. Lab solved (PortSwigger wykrywa wydany plik).

Liczba `../` z zapasem — obrazki leżą np. w `/var/www/images/`, a nadmiar `..` w korzeniu nie szkodzi.

---

## Przebieg

1. **Access the lab**, wejdź w produkt (żeby wygenerować request po obrazek).
2. `GET /image?filename=53.jpg` → **Send to Repeater** (Ctrl+R).
3. Podmień `53.jpg` → `../../../../../etc/passwd`, **Send**.
4. W Response pojawia się treść `/etc/passwd`. ✅

---

## 💥 Moje potknięcia

Tu poległem nie na traversalu, lecz na **podstawach Burpa** — zapisane na przyszłość (przyda się w każdej kolejnej labce):
- **Zakładka „HTTP history", nie „Intercept".** Intercept to kolejka wstrzymanych requestów; recon robisz w historii.
- **Intercept OFF** — inaczej Burp zatrzymuje *każdy* request (YouTube, reklamy z innych kart) i się w tym topisz.
- **„Open browser"** — wbudowana przeglądarka Burpa → czysta historia bez szumu z reszty internetu.
- **Musisz kliknąć w produkt**, żeby wygenerować request po obrazek — historia sama się nie zapełni.
- **Filtr historii** może ukrywać część MIME (obrazki) — zawężaj po hoście laba.

---

## Czego ten lab dokłada

Najprostszy wariant (odpowiednik gołego `'` w SQLi albo braku checka w access control 01). Utrwala **mechanikę `..`** i kluczowe rozróżnienie **read ≠ execute** (traversal czyta; `ls`/komendy to OS command injection). Dalej w temacie dochodzą obejścia filtrów: absolutna ścieżka, `....//`, URL/podwójny encode, null byte, wymuszone rozszerzenie.
