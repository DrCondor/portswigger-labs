# Authentication vulnerabilities

> Authentication to warstwa „kim jesteś" — drzwi wejściowe, które access control zakłada jako działające. Web stoi głównie na haśle, więc temat = atakowanie logowania hasłowego: każdy krok logowania to potencjalny **wyciek informacji** (enumeration) albo **niewyegzekwowana bramka** (2FA bypass, brak rate-limitu).

---

## O co chodzi (koncept)

Authentication opiera się na trzech filarach:

| Faktor | Co to | Przykład |
|---|---|---|
| **Something you know** | wiedza | hasło, PIN |
| **Something you have** | posiadanie | telefon (SMS), token, klucz U2F |
| **Something you are** | cecha | odcisk palca, twarz |

Większość webu stoi na filarze 1 (hasło); 2FA dokłada filar 2. Podatności biorą się stąd, że mechanizm logowania **zdradza więcej, niż powinien**, albo **nie egzekwuje tego, co obiecuje**.

---

## Jak to działa (mechanizm) — dwa główne wektory

### 1. Username enumeration — aplikacja jako oracle

Aplikacja **zdradza, czy dany username istnieje**, przez różnicę w zachowaniu:
- zły username → `"Invalid username"`
- dobry username, złe hasło → `"Incorrect password"`

Skoro odpowiedź jest inna, logowanie **odpowiada na pytanie „czy ten user istnieje?" jeszcze przed znajomością hasła** — to jest **oracle** (wyrocznia).

> ⭐ **Oracle zamienia mnożenie w dodawanie.** Zamiast 100 userów × 100 haseł = 10 000 prób, robisz 100 (znajdź usera) + 100 (znajdź hasło) = **200 prób**.

Różnica bywa subtelna — nie tylko inny tekst. Szukaj w: **status code**, **długości odpowiedzi (Length)**, **czasie odpowiedzi**, albo drobiazgu w treści (kropka/spacja/wielkość liter). `302` przy logowaniu = zwykle sukces (redirect), `200` = porażka.

### 2. Niewyegzekwowana bramka (np. 2FA bypass)

Gdy logowanie jest wieloetapowe (hasło → kod 2FA), dziura pojawia się, gdy serwer **za wcześnie** nadaje stan „zalogowany". Po kroku z hasłem sesja już należy do usera, a ekran kodu to tylko formularz. Jeśli strony logged-in-only nie sprawdzają „czy 2FA ukończone" — wchodzisz na docelową stronę wprost, pomijając kod. To **broken access control osadzony w przepływie 2FA**.

---

## Ogólny schemat ataku

**Brute-force (Burp Intruder):**
> przechwyć `POST /login` → Send to Intruder → Sniper → sortuj wyniki po **Status + Length** → znajdź **odstający wiersz**.

To ta sama „szukaj sieroty w kolumnie Length" mechanika, co przy SSRF (skan sieci) i generalnie przy enumeracji.

**Metodyka wieloetapowa (2FA / wizard):**
> rekonesans happy-path na **własnym** koncie (poznaj URL-e i kroki) → powtórz na koncie ofiary z **pominięciem bramki** (wejdź wprost na stronę końcową).

---

## Jak się przed tym bronić

**Przeciw enumeracji:**
1. **Generyczny komunikat** — identyczny co do bajta dla „zły user" i „złe hasło": *„Invalid username or password"*.
2. **Ten sam status / czas / długość** w obu przypadkach (uwaga też na timing i redirecty).
3. **Brute-force protection** — rate-limiting, lockout, CAPTCHA, MFA.

**Przeciw bypassowi bramki:**
> Stan „zalogowany" wolno nadać dopiero po ukończeniu **WSZYSTKICH** kroków. Po haśle user ma tylko **pending session**; każda chroniona strona weryfikuje w sesji flagę „2FA completed". Bramką jest serwer, nie ekran/redirect.

---

## Słowniczek

- **Authentication vs authorization** — „kim jesteś" vs „co ci wolno".
- **Username enumeration** — wykrycie ważnych loginów po różnicy w zachowaniu aplikacji.
- **Oracle** — mechanizm, który mimochodem potwierdza/zaprzecza hipotezę atakującego.
- **Brute-force (Intruder Sniper)** — automatyczne przemiatanie listy w jednej pozycji §.
- **`302` przy logowaniu** = zwykle sukces (redirect); `200` = porażka.
- **2FA / MFA** — drugi (kolejny) czynnik uwierzytelnienia.
- **Pending vs authenticated session** — sesja „w trakcie logowania" nie powinna dawać dostępu do niczego chronionego.
- **Generic error message** — obrona przed enumeracją.

> **Burp Intruder w Community Edition jest dławiony (~1 req/s)** — ~100 pozycji = ~1–2 min. To celowe ograniczenie, nie błąd.

---

## Laby w tym temacie

- [01 — username enumeration via different responses](01-username-enumeration-different-responses.md) — oracle po długości odpowiedzi (Length 3354 vs 3352), potem brute-force hasła (302 = sukces).
- [02 — 2FA simple bypass](02-2fa-simple-bypass.md) — sesja nadana po haśle; wejście wprost na `/my-account?id=carlos`, pomijając kod.

---

## Mindset / co się przenosi

Na każdym formularzu logowania/rejestracji/resetu pytaj: **„czy aplikacja zachowuje się inaczej dla ważnego vs nieważnego usera?"** (tekst, status, długość, czas). Jeśli tak — masz oracle.

Gdy auth jest wieloetapowe, pytaj: **„czy serwer faktycznie sprawdza ukończenie każdego kroku, czy tylko pokazuje kolejne ekrany?"** i spróbuj **przeskoczyć** krok. Dalej w temacie (poza Apprentice): enumeracja timingiem, obejścia lockoutu, luki w resecie hasła (password reset poisoning).
