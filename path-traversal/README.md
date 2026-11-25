# Path traversal (directory traversal)

> Aplikacja czyta plik z dysku po nazwie, którą jej podasz (`read(baza + filename)`). Wsadzasz `../`, wychodzisz poza katalog bazowy i czytasz dowolny plik, do którego proces ma dostęp. To prymityw **READ, nie execute** — potężny (configi, klucze, kod źródłowy), ale ograniczony do odczytu.

---

## O co chodzi (koncept)

Aplikacja buduje ścieżkę do pliku ze stałej bazy i Twojego inputu:

```
<img src="/image?filename=53.jpg">
serwer:  read("/var/www/images/" + filename)
              └── stała baza ──┘   └ Twój input
```

Bez sanityzacji wsadzasz `../` i **wychodzisz poza katalog bazowy** aż do korzenia, potem schodzisz, gdzie chcesz:

```
filename = ../../../../../etc/passwd
read("/var/www/images/../../../../../etc/passwd")  →  /etc/passwd
```

`..` to „katalog wyżej" — ta sama mechanika, co w normalizacji ścieżek przy web cache deception. Tu używasz jej do czytania plików z dysku serwera.

---

## Jak to działa (mechanizm)

Ścieżka w systemie plików ma hierarchię katalogów. `..` cofa o poziom:
```
/var/www/images/../../../etc/passwd
       cofnij×3 do korzenia, potem /etc/passwd
```

**Nadmiar `..` nie szkodzi** — `..` w korzeniu (`/`) zostaje w korzeniu. Dlatego jak nie wiesz, ile poziomów się cofnąć, dajesz z zapasem: `../../../../../etc/passwd`.

### ⭐ Kluczowe rozróżnienie: READ ≠ EXECUTE

Path traversal **tylko czyta plik**:
- ❌ **Nie odpalisz `ls`/`cat`/komend** — to nie shell, tylko funkcja czytająca plik.
- ❌ **Nie wylistujesz katalogów** — czytasz tylko plik, którego ścieżkę znasz lub zgadniesz.
- ✅ **Odczytasz dowolny plik**, do którego proces ma uprawnienia: `/etc/passwd`, configi, klucze SSH (`~/.ssh/id_rsa`), kod źródłowy.

Nazwa klasy: **arbitrary file read / file disclosure** (czasem **LFI** — Local File Inclusion). Chcesz wykonywać komendy → to **OS command injection**, osobna podatność (inny mechanizm).

---

## Ogólny schemat ataku

**Znajdź parametr sklejany ze ścieżką pliku (`filename=`, `file=`, `path=`, `doc=`) → wyjdź przez `../` → przeczytaj cel.**

`/etc/passwd` to kanoniczny **kanarek** odczytu na Linuksie: zawsze istnieje, czytelny dla wszystkich, rozpoznawalny format (`root:x:0:0:...`), a to tylko odczyt — więc bezpieczny test „czy się da". Delikatniejszy probe na to, czy `..` jest w ogóle przetwarzane: `filename=../images/53.jpg` — jak wróci ten sam obrazek, traversal działa.

---

## Jak się przed tym bronić

- **Nie buduj ścieżki z surowego inputu.** Jeśli musisz, po sanityzacji **zweryfikuj ścieżkę kanoniczną** (`File.expand_path`) i sprawdź, że wciąż jest wewnątrz dozwolonego katalogu.
- Najlepiej: nie pozwalaj wybierać pliku po nazwie z dysku — mapuj dozwolone ID na ścieżki po stronie serwera (whitelista).

---

## Słowniczek

- **path traversal / directory traversal** — wyjście poza katalog bazowy przez `../`.
- **arbitrary file read / file disclosure / LFI** — nazwy klasy podatności.
- **`/etc/passwd`** — kanoniczny kanarek odczytu na Linuksie.
- **injection point** — parametr sklejany ze ścieżką pliku (`filename=`).
- **read ≠ execute** — path traversal czyta; OS command injection wykonuje.

---

## Laby w tym temacie

- [01 — file path traversal, simple case](01-file-path-traversal-simple-case.md) — brak zabezpieczeń, goły `../` → `/etc/passwd`.

*(PortSwigger ma dalej obejścia filtrów — na razie zrobiony simple case w ramach ścieżki Server-side.)*

---

## Mindset / co się przenosi

Path traversal = **prymityw odczytu dowolnego pliku**. W realu potężny (sekrety, klucze, kod), ale zawsze *czytasz*, nie *wykonujesz*. Mechanika `..` (cofnij katalog) wraca w normalizacji ścieżek (web cache deception) i ogólnie przy canonicalization bypassach.

Dalej w temacie: obejścia, gdy simple case nie przejdzie — absolutna ścieżka, zagnieżdżony traversal odporny na stripowanie (`....//`), URL/podwójny encode (`%2e%2e%2f`, `%252e`), wymuszony prefiks katalogu, null byte (`%00`), obowiązkowe rozszerzenie.
