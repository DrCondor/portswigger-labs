# Server-Side Request Forgery (SSRF)

> Zamiast atakować aplikację jako klient, **zmuszasz serwer, żeby to on wysłał żądanie** tam, gdzie wskażesz (parametr z URL-em: `stockApi=`, `url=`, `webhook=`…). Serwer staje się twoim posłańcem — i sięga tam, gdzie ty z internetu nie dojdziesz: loopback, sieć wewnętrzna, inne porty za firewallem.

---

## O co chodzi (koncept)

Dotąd atakowałeś aplikację **jako klient**: ty wysyłałeś żądanie, serwer ci (nie)ufał. SSRF odwraca kierunek:

> **Zmuszasz serwer, żeby to on wysłał żądanie do celu, który ty wskażesz.**

Aplikacja ma funkcję, która z założenia sama strzela gdzieś po sieci (pobiera URL, sprawdza stan magazynu w innym systemie, waliduje webhook, ściąga obrazek). Ty podmieniasz **cel** tego żądania — serwer staje się twoim **proxy**.

**Analogia:** prosisz recepcjonistę w firmie, żeby zadzwonił pod numer wewnętrzny, do którego ty z zewnątrz się nie dodzwonisz. On dzwoni „ze środka", a system po drugiej stronie ufa, bo telefon przyszedł z wewnętrznej centrali.

---

## Jak to działa (mechanizm): zaufanie topologiczne

Cała groźność SSRF stoi na jednym: **wiele systemów ufa żądaniu po jego *pochodzeniu* (skąd), a nie po *tożsamości* (kto).**

| | Ty z internetu | Serwer aplikacji |
|---|---|---|
| Dostęp do `localhost` / sieci wewnętrznej | ❌ firewall blokuje | ✅ jest **wewnątrz** |
| Jak widzi go `/admin` | obcy, nieautoryzowany | „swój", z zaufanego miejsca |

Panel admina często zakłada: „skoro request leci z `127.0.0.1` / sieci lokalnej, to ktoś zaufany". SSRF to pochodzenie **fałszuje**, bo request fizycznie wychodzi z serwera. To **access control oparty na lokalizacji sieciowej zamiast na tożsamości** — i właśnie to łamiesz.

Dlaczego serwery tak ufają lokalnym żądaniom:
- **check uprawnień siedzi w komponencie *przed* aplikacją** (reverse proxy / gateway) — połączenie zwrotne serwera do siebie omija ten front,
- **tryb disaster-recovery** — dostęp admina bez logowania „jeśli z local machine",
- **panel admina na innym porcie**, nieosiągalnym z zewnątrz,
- back-endy „chronione topologią sieci" mają często **słabszy security posture** (wrażliwe funkcje bez autentykacji).

---

## Ogólny schemat ataku

**Znajdź parametr z URL-em → wskaż nim adres wewnętrzny → serwer sięga tam jako „zaufany" → czytasz odpowiedź.**

- **Cel na samym serwerze:** `localhost` / `127.0.0.1` (uwaga: to **serwer aplikacji**, nie twój komputer).
- **Cel w sieci wewnętrznej, adres nieznany:** SSRF + **Intruder** = skaner sieci — przelatujesz zakres adresów/portów i szukasz **odstającego** (Status/Length) → tam coś żyje. To ta sama „szukaj sieroty w kolumnie Length" mechanika co przy username enumeration.

---

## Jak się przed tym bronić

- **Nie podejmuj decyzji o dostępie na podstawie pochodzenia żądania** (lokalne ≠ zaufane) — wymagaj tożsamości/autentykacji nawet od wewnętrznych połączeń.
- **Whitelista** dozwolonych hostów/URL-i dla funkcji pobierającej; blokuj loopback i prywatne zakresy IP.
- Uwaga na **obejścia blacklisty** (alternatywne zapisy IP, redirecty) — blacklista `localhost` to słaba obrona.

---

## Słowniczek

- **SSRF** — zmuszenie serwera do wysłania żądania do celu wskazanego przez atakującego.
- **Loopback / `localhost` / `127.0.0.1`** — adres „do samego siebie"; z zewnątrz nieosiągalny, z serwera tak.
- **Zaufanie topologiczne** — ufanie żądaniu po *pochodzeniu* zamiast po *tożsamości*; fundament, który SSRF łamie.
- **In-band / returned SSRF** — serwer odsyła treść pobranego URL-a (widzisz wynik). Przeciwieństwo: **blind SSRF** (nic nie wraca — Burp Collaborator).
- **Prywatny / nie-routowalny IP** — `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`; tylko z wnętrza sieci.
- **Port scanning przez SSRF** — podatny serwer jako proxy do mapowania żywych hostów/portów.

> **Community dławi Intrudera ~1 req/s** — skan 255 adresów to ~4 min, nie zawieszenie.

---

## Laby w tym temacie

- [01 — basic SSRF against the local server](01-basic-ssrf-local-server.md) — `stockApi=http://localhost/admin`, panel w odpowiedzi, Delete carlosa.
- [02 — basic SSRF against another back-end system](02-basic-ssrf-another-backend.md) — SSRF jako skaner: Intruder na ostatnim oktecie `192.168.0.X:8080`, sierota = żywy host.

---

## Mindset / co się przenosi

Sygnał „to może być SSRF": aplikacja przyjmuje **URL / host / adres** jako input i coś z nim robi po stronie serwera (`stockApi=`, `url=`, `feed=`, `callback=`, `webhook=`, ładowanie obrazka po URL).

SSRF to często **wytrych do broken access control** — panel sam w sobie bywa chroniony, ale ufa „lokalnemu" pochodzeniu. Dalej w temacie: obejścia filtrów na `localhost` (alternatywne zapisy IP, redirecty) i **blind SSRF** (Collaborator).
