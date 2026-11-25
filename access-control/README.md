# Access control (authorization)

> Authentication mówi „kim jesteś", a access control — „czy **wolno ci** to zrobić". Broken access control to sytuacja, w której aplikacja wie, kim jesteś, a mimo to nie sprawdza, czy masz prawo do danej akcji lub zasobu. OWASP Top 10 #1.

---

## O co chodzi (koncept)

Trzy warstwy, które brzmią podobnie, ale to **nie to samo**:

| Warstwa | Pytanie | Rails-owo |
|---|---|---|
| **Authentication** | „Kim jesteś?" | login/hasło, sesja |
| **Session management** | „Czy to nadal ty?" | cookie `session=...` |
| **Authorization / Access control** | „Czy **wolno ci** to zrobić?" | `require_admin`, Pundit/CanCanCan |

**Broken access control** = zawodzi ta trzecia warstwa. Dwie pierwsze działają (wiadomo, kim jesteś), ale aplikacja nie weryfikuje **uprawnień** do konkretnego zasobu lub akcji.

**Analogia:** masz ważny bilet na pociąg (authentication — wpuszczono cię na peron), ale nikt nie sprawdza, czy wchodzisz do swojego przedziału, czy do kabiny maszynisty. Access control to ten brakujący konduktor przy drzwiach każdego przedziału.

---

## Jak to działa (mechanizm) — rodzaje broken access control

Podatność ma kilka smaków, od najprostszego do subtelnego:

**1. Unprotected functionality** — wrażliwa funkcja po prostu **nie ma żadnej bramki**. Atak = znaleźć URL. Bywa „ukryty" przez:
- `robots.txt` (który *ogłasza* wrażliwe ścieżki, nie chroni ich),
- losową nazwę endpointu (security through obscurity),
- link schowany w kliencie (`if(isAdmin)` w JS — kod i tak leci do każdego).

**2. Parameter-based (vertical escalation)** — rola/uprawnienia trzymane w danych kontrolowanych przez klienta (cookie `Admin=false`, hidden field, `?admin=true`). Klient je kontroluje → podmienia → dostaje wyższe uprawnienia.

**3. IDOR / horizontal escalation** — endpoint bierze `id` zasobu **z inputu** (`?id=123`, `?userId=GUID`) i oddaje go bez sprawdzenia, czy należy do zalogowanego usera. Dostęp „w bok", do danych innego usera. W nazewnictwie OWASP API: **BOLA** (#1).

**Dwa kierunki eskalacji:**
- **Vertical** — z niższego poziomu na wyższy (user → admin).
- **Horizontal** — do zasobów innego usera na tym samym poziomie. Horizontal **zamienia się w vertical**, gdy „innym userem" jest admin (przejmujesz jego konto → jego uprawnienia).

### Powtarzający się motyw: fałszywa „ochrona"

Przez cały temat wraca jeden wzorzec: **check może istnieć, ale być zastąpiony czymś, co ochroną nie jest**:
- **security through obscurity** — ukrycie (robots.txt, losowa nazwa, GUID) ≠ zabezpieczenie,
- **client-side control** — ukryty przycisk / `if` w JS / maskowanie `type=password` = kosmetyka w przeglądarce, którą atakujący kontroluje,
- **check w złym miejscu** — chroniona *strona* panelu, ale nie *akcja* pod nią (strona ≠ akcja).

---

## Ogólny schemat ataku

**Enumeracja ścieżki (robots.txt → źródło/JS → API…) → wywołaj endpoint bezpośrednio → brak/wadliwy check = broken access control.**

Odruchy na nowym targecie:
1. `GET /robots.txt`, `GET /sitemap.xml` — właściciel często sam wskazuje wrażliwe miejsca.
2. Czytaj **surowe źródło** (View Source / Burp HTTP history), nie tylko wyrenderowany DOM — ukryte linki, hasła w `value`, ścieżki w JS.
3. Po zalogowaniu pytaj: **„gdzie aplikacja trzyma info, kim/czym jestem — i czy mogę to podmienić?"** (cookie, param, hidden field).
4. Widzisz `?id=` / `?userId=` → **„a co, jak wstawię cudzą wartość?"**. Jeśli id jest losowe (GUID) — poluj na miejsce, gdzie aplikacja je zdradza.

---

## Jak się przed tym bronić

Dwie żelazne zasady:

**1. Decyzje o dostępie tylko na danych, których klient nie może podmienić** — rola z sesji/bazy, nigdy z cookie/parametru.

**2. Check na KAŻDYM wrażliwym endpoincie — per-akcja i per-obiekt.**

```ruby
class Admin::UsersController < ApplicationController
  before_action :require_admin            # per-akcja (vertical)

  def destroy
    user = User.find(params[:id])
    head :forbidden unless user == current_user || current_user.admin?  # per-obiekt (horizontal/IDOR)
    user.destroy
  end

  private
  def require_admin
    head :forbidden unless current_user&.admin?
  end
end

# IDOR fix w skrócie: bierz zasób z sesji, nie z id od klienta
@user = current_user              # zamiast User.find(params[:id])
```

Ani losowa nazwa, ani GUID, ani ukryty przycisk, ani maskowanie hasła **nie są** zabezpieczeniem — to tylko utrudnienia. Prawdziwa ochrona to autoryzacja po stronie serwera.

---

## Słowniczek

- **Access control / authorization** — „czy wolno ci to zrobić" (≠ authentication „kim jesteś").
- **Broken access control** — brak/wadliwy check uprawnień; OWASP #1.
- **Unprotected functionality** — wrażliwa funkcja bez bramki; atak = znaleźć URL.
- **Privilege escalation** — zdobycie uprawnień, których nie powinieneś mieć. **Vertical** (user→admin) / **Horizontal** (do zasobu innego usera).
- **IDOR** (Insecure Direct Object Reference) / **BOLA** (Broken Object Level Authorization) — odwołanie do obiektu przez id z inputu, bez authz per-obiekt.
- **Security through obscurity** — „bezpieczeństwo" oparte na ukrywaniu (robots.txt, losowa nazwa, GUID); antywzorzec.
- **Client-side access control** — decyzja o dostępie w przeglądarce (JS/ukrywanie UI); zawsze do obejścia.
- **Chaining** — złożenie kilku drobiazgów (wyciek id + IDOR + disclosure) w pełny atak.
- **Strona ≠ akcja** — zabezpieczenie widoku panelu to nie to samo co zabezpieczenie endpointu akcji.

---

## Laby w tym temacie

- [01 — unprotected admin functionality](01-unprotected-admin-functionality.md) — panel bez checka, ścieżka w `robots.txt`.
- [02 — unprotected admin, unpredictable URL](02-unprotected-admin-unpredictable-url.md) — ścieżka panelu wyciekła w JS strony głównej (`if(isAdmin)`).
- [03 — user role controlled by request parameter](03-user-role-controlled-by-request-parameter.md) — rola w sfałszowalnym cookie `Admin=false` (vertical).
- [04 — user ID controlled by parameter, unpredictable GUIDs](04-user-id-unpredictable-guid.md) — IDOR; GUID carlosa wyciekł w module bloga (horizontal + chaining).
- [05 — user ID controlled by parameter with password disclosure](05-user-id-password-disclosure.md) — IDOR + hasło admina w `value` → horizontal→vertical.

---

## Mindset / co się przenosi

Rdzeń całej klasy: **check może istnieć, ale być w złym miejscu, opierać się na danych klienta, albo być zastąpiony fałszywą „ochroną" (obscurity / maskowanie / losowe id).** Prawdziwa obrona = autoryzacja po stronie serwera, per-akcja i per-obiekt.

Motyw „nie ufaj klientowi" wraca wszędzie: w authentication (2FA enforcement), file upload (`Content-Type`), SQLi (walidacja front-endowa). Access control to miejsce, gdzie poznajesz go najczyściej.
