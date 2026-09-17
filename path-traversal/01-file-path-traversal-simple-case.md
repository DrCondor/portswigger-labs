# Lab 01 — File path traversal, simple case

- **Kategoria:** Server-side vulnerabilities → Path traversal (directory traversal)
- **Poziom:** Apprentice
- **Data:** 2026-09-17
- **Status:** ✅ rozwiązany
- **Narzędzia:** Burp Suite Community (Proxy + Repeater)
- **Ścieżka:** część learning path „Server-side vulnerabilities" (fundamenty przed Web LLM attacks)

---

## Cel labu

Podatność path traversal siedzi w **wyświetlaniu obrazków produktów**. Trzeba **odczytać zawartość `/etc/passwd`**.

---

## Koncept: path traversal = arbitrary file read

Aplikacja czyta plik z dysku na podstawie nazwy, którą jej podasz:

```
<img src="/image?filename=53.jpg">
serwer:  read("/var/www/images/" + filename)
              └── stała baza ──┘   └ Twój input
```

Bez sanityzacji wsadzasz `../` i **wychodzisz poza katalog bazowy** aż do korzenia, potem schodzisz gdzie chcesz:

```
filename = ../../../../../etc/passwd
read("/var/www/images/../../../../../etc/passwd")  →  /etc/passwd
```

**To ta sama mechanika `..`, co przy [[03-origin-normalization]] / [[04-cache-normalization]] w cache deception** — „cofnij katalog wyżej". Tam myliłeś cache/origin; tu czytasz pliki z dysku serwera.

---

## ⭐ Kluczowe rozróżnienie: READ, nie EXECUTE

Path traversal to prymityw **„przeczytaj plik"** — i tylko tyle:

- ❌ **Nie odpalisz `ls`, `cat`, żadnej komendy** — to nie shell, tylko funkcja czytająca plik.
- ❌ **Nie wylistujesz katalogów** — czytasz tylko plik, którego ścieżkę **znasz albo zgadniesz**.
- ✅ **Odczytasz dowolny plik**, do którego proces ma uprawnienia: `/etc/passwd`, configi, klucze SSH (`/home/user/.ssh/id_rsa`), kod źródłowy apki.

Nazwa: **arbitrary file read / file disclosure** (czasem: LFI — Local File Inclusion). Groźne (sekrety, configi), ale **ograniczone do odczytu** — nic nie wykonujesz.

> Chcesz `ls` / wykonywać komendy → to **OS command injection**, osobna podatność dalej w tej ścieżce. Inny mechanizm.

**„Jak sprawdzić, czy się da" bez ryzyka:** `/etc/passwd` JEST tym testem — zawsze istnieje, czytelny dla wszystkich, rozpoznawalny format (`root:x:0:0:...`), a to **tylko odczyt**, więc nic nie zepsujesz. Delikatniejszy probe na to, czy `..` jest przetwarzane: `filename=../images/53.jpg` → jak wróci ten sam obrazek, traversal działa.

---

## Mechanizm tego labu (simple case)

```
GET /image?filename=../../../../../etc/passwd HTTP/2
```
→ Response: treść `/etc/passwd` zamiast obrazka. Lab solved (PortSwigger wykrywa wydany plik).

**„Simple case" = zero zabezpieczeń** → goły `../` przechodzi, bez enkodowania. Obejścia (absolutna ścieżka, `....//`, URL-encode, null byte) to kolejne labki tego działu.

---

## Kroki

1. **Access the lab**, wejdź w produkt.
2. Znajdź request ładujący obrazek: `GET /image?filename=53.jpg` → **Send to Repeater**.
3. Podmień `53.jpg` → `../../../../../etc/passwd`, **Send**.
4. W Response (Raw/Pretty) pojawia się `root:x:0:0:root:/root:/bin/bash` itd. ✅

---

## Pułapki / lekcje

### Path traversal
- **Liczba `../`** — obrazki leżą np. w `/var/www/images/`, trzeba się cofnąć kilka poziomów. **Nadmiar `..` nie szkodzi** (`..` w korzeniu zostaje w korzeniu) → jak nie wchodzi, dawaj więcej: `../../../../../etc/passwd`.
- **To odczyt, nie egzekucja** — patrz wyżej.

### ⭐ Workflow Burpa (przyda się w KAŻDEJ kolejnej labce)
Tu straciłem czas na podstawy Burpa, więc zapisuję na przyszłość:
- **Zakładka „HTTP history", nie „Intercept".** Intercept to kolejka wstrzymanych requestów; recon robisz w historii.
- **Intercept OFF** — inaczej Burp zatrzymuje *każdy* request (łapie YouTube, reklamy z innych kart) i się w tym topisz.
- **„Open browser"** — wbudowana przeglądarka Burpa. Otwieraj laba w niej → czysta historia, bez szumu z reszty internetu.
- **Musisz kliknąć w produkt**, żeby wygenerować request po obrazek — sama historia się nie zapełni bez ruchu.
- **Filtr historii** może ukrywać część MIME (obrazki) — zawężaj po hoście laba, żeby odsiać śmieci.
- `Ctrl+R` = Send to Repeater.

---

## Pojęcia do zapamiętania

- **path traversal / directory traversal** — wyjście poza katalog bazowy przez `../`
- **arbitrary file read / file disclosure / LFI** — nazwa klasy podatności
- **`/etc/passwd`** — kanoniczny „kanarek" odczytu na Linuksie (zawsze jest, czytelny, rozpoznawalny)
- **injection point** — parametr sklejany ze ścieżką pliku (`filename=`)
- **read ≠ execute** — path traversal czyta; OS command injection wykonuje

---

## Mindset

Path traversal = **prymityw odczytu dowolnego pliku**. W realu potężny: kradniesz configi, klucze, sekrety, kod źródłowy — ale zawsze *czytasz*, nie *wykonujesz*. Schemat: **znajdź parametr sklejany ze ścieżką → wyjdź przez `../` → przeczytaj cel**.

**Następne w temacie:** obejścia zabezpieczeń, gdy „simple case" nie przejdzie — absolutna ścieżka (`/etc/passwd`), zagnieżdżony traversal odporny na stripowanie (`....//`), URL/podwójny encode (`%2e%2e%2f`, `%252e`), wymuszony prefiks katalogu, null byte (`%00`), obowiązkowe rozszerzenie.
