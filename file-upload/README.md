# File upload vulnerabilities

> Serwer przyjmuje i zapisuje plik **bez wystarczającej walidacji** (nazwa/typ/zawartość/rozmiar). Najgorszy wariant: wgrywasz **plik ze skryptem server-side** (PHP/JSP…), a serwer jest skonfigurowany, żeby go **wykonać** — upload „obrazka" staje się **RCE** (zdalne wykonanie kodu).

---

## O co chodzi (koncept)

File upload vuln = serwer zapisuje przesłany plik na filesystemie, nie sprawdzając wystarczająco, czy to, co dostał, jest tym, czego oczekiwał. Skala skutków zależy od tego, **co da się wgrać i co serwer z tym robi**:
- plik przechodzi do publicznego katalogu → możliwy **web shell** (RCE),
- brak limitu rozmiaru → DoS,
- nazwa z path traversal → nadpisanie plików.

**Web shell** = złośliwy skrypt dający wykonywanie kodu/komend na serwerze przez zwykłe HTTP. Udany web shell = faktycznie pełna kontrola nad serwerem.

---

## Jak to działa (mechanizm): o wykonaniu decyduje konfiguracja serwera

Sam plik nic nie robi — o tym, czy kod się wykona, decyduje **konfiguracja serwera WWW**:

| Prosisz o… | Serwer robi… | Dostajesz… |
|---|---|---|
| `avatar.png` | oddaje bajty pliku | statyczny obrazek |
| `shell.php` | **uruchamia interpreter PHP** na pliku | **wynik wykonania kodu** |

PHP to dziedzictwo modelu CGI: „jeden plik = jeden endpoint, interpretowany na żądanie". Apache/Nginx typowo mają regułę „każdy `.php` w katalogu webowym → przepuść przez PHP". Dlatego PHP jest modelową ofiarą tych labów — nie bo „hakerski", tylko bo **domyślnie wykonywalny per-plik**.

> **To samo dałoby się w Ruby/Perl/ASP — gdyby serwer był tak skonfigurowany (CGI, mod_ruby).** Rails standardowo NIE działa w modelu „plik=strona" (`public/` serwuje statycznie, routing przez `routes.rb`), więc wrzucony `.rb` leży jak martwy tekst. Podatność siedzi w **relacji upload + konfiguracja wykonywania plików**, nie w języku.

Web shell w PHP (model jak `.erb`: kod tylko wewnątrz `<?php ?>`, reszta leci dosłownie):
```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```
`echo` ~ `puts`/`print`, `file_get_contents()` ~ `File.read`.

---

## Ogólny schemat ataku

**Wgraj wykonywalny plik → znajdź URL, pod którym serwer go udostępnia → poproś o niego → serwer odpala kod → czytasz wynik w odpowiedzi.**

Dowód wykonania: jeśli odpowiedź zawiera **wynik** skryptu (np. sekret), a nie jego kod źródłowy, interpreter zadziałał. Np. `Content-Length` = długość sekretu, nie długość kodu PHP.

Gdy upload ma walidację — obchodzisz ją (patrz: labki od 02 w górę): kłamiesz w `Content-Type`, obchodzisz blacklistę rozszerzeń (`.phtml`/`.php5`), wrzucasz `.htaccess`, chowasz PHP w metadanych obrazka (polyglot), null byte, path traversal w nazwie.

---

## Jak się przed tym bronić

- **Waliduj zawartość, nie deklarację** — sprawdzaj magic bytes / faktyczny typ pliku, nie `Content-Type` od klienta ani samo rozszerzenie.
- **Serwuj uploady z katalogu, w którym serwer niczego nie wykonuje** (osobny storage / CDN, `Content-Disposition: attachment`).
- **Generuj nazwy plików po stronie serwera** (nie ufaj nazwie od klienta — path traversal, podwójne rozszerzenia).
- Whitelista rozszerzeń + limit rozmiaru.

---

## Słowniczek

- **File upload vulnerability** — przyjęcie i zapis pliku bez wystarczającej walidacji (nazwa/typ/zawartość/rozmiar).
- **Web shell** — skrypt na serwerze dający wykonywanie kodu/komend przez HTTP; udany = pełna kontrola.
- **RCE (Remote Code Execution)** — zdalne wykonanie kodu; najgroźniejszy skutek uploadu.
- **Server-side script execution** — serwer wykonuje `.php`/`.jsp`/… zamiast serwować jako statyk; o tym decyduje konfiguracja, nie plik.
- **MIME type / `Content-Type` (per-part)** — deklaracja typu danych pola w `multipart/form-data`; ustawiana przez klienta, więc niewiarygodna.
- **Magic bytes / file signature** — pierwsze bajty pliku identyfikujące prawdziwy typ (np. `FF D8 FF` = JPEG); poprawna walidacja patrzy tu.

---

## Laby w tym temacie

- [01 — RCE via web shell upload](01-rce-via-web-shell-upload.md) — zero walidacji; `shell.php` → `GET /files/avatars/shell.php` → sekret.
- [02 — Content-Type restriction bypass](02-content-type-restriction-bypass.md) — serwer ufa `Content-Type`; podmiana na `image/jpeg` przy `filename="shell.php"`.

*(PortSwigger ma 9 labek — dalej: blacklisty rozszerzeń, `.htaccess`, path traversal w nazwie, polygloty.)*

---

## Mindset / co się przenosi

Sygnał „to może być file upload → RCE": jest **upload pliku** (avatar, załącznik) + podejrzenie, że serwer **wykonuje** pewne rozszerzenia. Rdzeń ataku zawsze ten sam: **dostarcz wykonywalny plik tam, gdzie serwer go odpali** — zmienia się tylko, które zabezpieczenie trzeba oszukać.

Szersza zasada: obrona oparta na danych od klienta (`Content-Type`, nazwa) to kosmetyka — ten sam motyw co cookie `Admin=false` (access control) i walidacja front-endowa (SQLi login bypass).
