# SQL injection (SQLi)

> Aplikacja skleja zapytanie SQL z twojego inputu jak zwykły tekst. Jeśli wstrzykniesz w to **cudzysłów**, twoje dane przestają być „wartością" i stają się **kodem SQL**, który baza wykona. Stąd: podgląd cudzych danych, logowanie bez hasła, a w cięższych wariantach przejęcie serwera.

---

## O co chodzi (koncept)

Aplikacje webowe trzymają dane w bazie i pytają o nie zapytaniami SQL. Problem zaczyna się, gdy aplikacja **buduje to zapytanie przez sklejanie tekstu** z danymi od użytkownika:

```
"SELECT * FROM products WHERE category = '" + input + "'"
```

Dla programisty `input` to „nazwa kategorii". Dla silnika bazy to po prostu **fragment tekstu zapytania**. Jeśli użytkownik wpisze coś, co silnik zinterpretuje jako **składnię SQL**, to ta składnia zostanie wykonana — bo baza nie odróżnia „danych, które wkleił programista" od „poleceń, które wkleił atakujący". Wszystko jest jednym stringiem.

**Analogia:** to jak dyktowanie komuś listu przez telefon. Mówisz „Drogi Panie, [treść]". Jeśli w miejsce treści podyktujesz „kropka. A teraz skasuj poprzednie zdanie i napisz co innego" — osoba pisząca nie wie, że to miała być *treść listu*, więc wykona twoje polecenie. SQLi to dokładnie to: twój input udaje treść, a jest instrukcją.

---

## Jak to działa (mechanizm)

### Kontekst danych vs kontekst kodu

Dane tekstowe w SQL żyją **wewnątrz cudzysłowów**: `'Gifts'`. Wszystko między `'` a `'` to dla silnika „goły tekst", nieważne jak groźnie wygląda. Siła SQLi bierze się z jednego ruchu:

> **Wpisujesz `'`, żeby zamknąć cudzysłów, który otworzyła aplikacja.** Od tego znaku wychodzisz z „kontekstu danych" (tekst) i wskakujesz w „kontekst kodu" (SQL). Dalej silnik czyta twój input jako polecenia.

To jest **esencja każdego injection** (SQL, OS command, XSS...): przeskoczyć z „jestem danymi" do „jestem kodem". W SQLi wytrychem jest `'`.

### Model mentalny: szablon z dziurą

Nie myśl o całym zapytaniu. Myśl o **szablonie, w który aplikacja wkleja cudzysłowy za ciebie**:

```
SELECT * FROM products WHERE category = '???' AND released = 1
                                        ↑   ↑
                            pisze APKA  |   | pisze APKA
                                   twój input
```

Ty kontrolujesz tylko `???`. Ale wstrzykując `'` możesz **domknąć** pierwszy cudzysłów i dopisać swój SQL. Jedyny kłopot: szablon dokleja po tobie `' AND released = 1` — zostaje osierocony cudzysłów, który wywaliłby składnię. Rozwiązujesz to komentarzem.

### Komentarz `--`

`--` w SQL oznacza „ignoruj resztę **tej linii**". Używasz go, żeby **odciąć ogon szablonu**, którego nie kontrolujesz (osierocony `'` i dalsze warunki):

```
category = 'Gifts'-- ' AND released = 1
                   └────── wykomentowane, silnik tego nie widzi ──────┘
```

Niuans między bazami:
- **PostgreSQL, Oracle, MSSQL:** samo `--` wystarczy.
- **MySQL:** `--` wymaga spacji po sobie (`-- `) albo użyj `#`.

---

## Ogólny schemat ataku

1. **Znajdź punkt wejścia** — parametr, który prawdopodobnie trafia do zapytania: filtr, wyszukiwarka, `id`, sortowanie, pole logowania.
2. **Testuj obecność** — wstrzyknij samo `'` i szukaj błędu / anomalii / zmiany w odpowiedzi. „Coś się zepsuło" = prawdopodobnie trafiłeś w string literał.
3. **Przejmij logikę** — zamknij string (`'`), dopisz swój SQL, wykomentuj resztę (`--`).

Powtarzalny rdzeń: **zamknij string (`'`) → dopisz swój SQL → wykomentuj resztę (`--`)**.

Typowe cele, zależnie od wariantu:
- **Podgląd ukrytych danych** — rozluźnij warunek `WHERE` (`OR 1=1`), żeby zwrócił więcej, niż powinien.
- **Logowanie bez hasła** — spraw, by zapytanie uwierzytelniające zwróciło wiersz admina bez znajomości hasła (`admin'--`).
- **Wyciąganie danych z innych tabel** — `UNION SELECT ...` (osobne laby).
- **Ślepe wnioskowanie** — gdy wyniku nie widać wprost: boolean (`AND 1=1` vs `AND 1=2`) albo czas (`sleep`).

---

## Jak się przed tym bronić

Fix jest jeden i prosty: **nigdy nie sklejaj SQL-a z danymi**. Przekaż wartość **osobno** jako parametr (prepared statement / parametryzowane zapytanie). Wtedy silnik dostaje strukturę zapytania i dane dwoma osobnymi kanałami — twój cudzysłów zostaje **daną** (nazwą kategorii), bo nigdy nie przechodzi przez parser jako część kodu.

Po railsowemu:
```ruby
# ❌ podatne — interpolacja do stringa SQL
User.where("category = '#{params[:category]}'")

# ✅ bezpieczne — bind param, wartość osobno
User.where(category: params[:category])
User.where("category = ?", params[:category])
```

To samo w każdym ekosystemie: `?`/`:named` placeholdery, nie konkatenacja.

---

## Słowniczek

- **SQL injection (SQLi)** — wstrzyknięcie składni SQL przez input sklejany w zapytanie; pozwala wyjść z danych do kodu.
- **Breaking out of the string** — zamknięcie cudzysłowu `'`, które przerzuca input z kontekstu danych do kontekstu kodu. Rdzeń ataku.
- **Komentarz `--`** — wycina resztę linii (osierocony `'` + dalsze warunki). MySQL: `-- ` ze spacją albo `#`.
- **`OR 1=1`** — zawsze-prawdziwy warunek; rozluźnia `WHERE` tak, że zwraca wszystko. Uwaga: jeśli dane trafiają też do `UPDATE`/`DELETE`, może zniszczyć dane.
- **Prepared statement / bind param** — przekazanie wartości osobno od zapytania; fix na SQLi.
- **UNION attack** — doklejenie `UNION SELECT` w celu wyciągnięcia danych z innych tabel.
- **Blind SQLi** — wariant bez widocznego wyniku; wnioskowanie po boolean/czasie.

---

## Laby w tym temacie

- [01 — WHERE clause, retrieval of hidden data](01-where-clause-hidden-data.md) — rozluźnienie filtra (`OR 1=1--`), żeby pokazać ukryte produkty.
- [02 — login bypass](02-login-bypass.md) — `admin'--` loguje jako administrator bez hasła.

---

## Mindset / co się przenosi

SQLi to **injection** — ta sama rodzina co OS command injection (wytrych `|`/`&`) i XSS (wytrych `<script>`). Gdy widzisz input, który prawdopodobnie ląduje w zapytaniu do bazy, odruch: **wstrzyknij `'` i patrz, czy coś się psuje**. Reszta tematu to warianty tego samego ruchu — zmienia się tylko, *co* robisz po wyjściu do kontekstu kodu.

Szersza zasada (ważna dla mnie jako deva): **struktura zapytania i dane to dwa różne kanały.** Gdy je zlejesz w jeden string, atakujący przejmuje strukturę. Trzymaj je osobno — zawsze bind params.
