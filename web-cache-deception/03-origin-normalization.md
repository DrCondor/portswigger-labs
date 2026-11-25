# Lab 03 — Exploiting origin server normalization for web cache deception

| | |
|---|---|
| **Poziom** | Practitioner |
| **Status** | ✅ rozwiązany |
| **Data** | 2026-09-05 |
| **Narzędzia** | Burp Community (Proxy + Repeater), exploit server |

> 📖 **Teoria tej klasy podatności: [README tematu](README.md).** Tu tylko ten konkretny lab.

---

## Cel

Wykraść **API key carlosa**, tym razem przez **rozbieżność w normalizacji ścieżki** (path traversal z zakodowanym slashem).

---

## Trik tego labu

Rozbieżność nie jest w sufiksie ani delimiterze, tylko w tym **jak bardzo każdy komponent normalizuje ścieżkę** (dekoduje `%2f`, rozwija `..`).

Payload: **`/resources/..%2fmy-account`**
- **origin** (normalizuje): dekoduje `%2f`→`/`, rozwija `..` → `/my-account` → konto carlosa,
- **cache** (nie normalizuje): widzi `/resources/...` → reguła **static directory** → zapisuje.

### ⭐ Dlaczego slash MUSI być zakodowany (`%2f`)

To cała istota triku. Ten sam string, dwa cięcia:

**Cache (NIE dekoduje `%2f` przed cięciem):**
```
/resources/..%2fmy-account
 └ "resources"
 └ "..%2fmy-account"   ← JEDEN segment (%2f to zwykły znak)
```
Brak samodzielnego `..` → nic do rozwinięcia → widzi `/resources/<coś>` → **static directory → cachuję**.

**Origin (NAJPIERW dekoduje `%2f`→`/`, potem tnie):**
```
/resources/../my-account
 └ "resources" └ ".." (kasuje resources) └ "my-account"
→ /my-account → prywatna strona z API key
```

Dlaczego nie zwykły `/`? Bo **przeglądarka sama rozwinęłaby `..` zanim wyśle** (poleciałoby gołe `/my-account`, cache nie zobaczy `/resources`), a do tego zniknęłaby rozbieżność (oba rozwinęłyby identycznie). `%2f` to **klin** chowający granicę `..` przed przeglądarką i cache'em — rozwija go dopiero origin.

---

## Przebieg

**Rekon:**
1. **Czy origin normalizuje?** `GET /aaa%2f..%2fmy-account` — śmieciowy `aaa` tylko po to, żeby `..` miał co skasować. Wróci konto (200 + API key) **tylko** jeśli origin zdekodował `%2f`, rozwinął `..` i wylądował na `/my-account`. Tu: 200 ✅.
2. **Czy cache trzyma `/resources`?** `GET /resources/js/tracking.js` 2× → `miss`→`hit`, `max-age=30` ✅.
3. **Dry run payloadu** → `GET /resources/..%2fmy-account` → wraca konto + `miss`→`hit` → potwierdza regułę **katalogu** (payload nie ma `.js`).

Delivery jak w [README](README.md); payload:
```
/resources/wcd/..%2f..%2fmy-account   ← świeża ścieżka
```

### ⭐ Trik: świeża ścieżka bez zmiany celu

Nie ma „nazwy pliku" do podmiany (`wcd1.js`→`wcd2.js`). Świeży klucz w cache robisz **dokładając śmieciowy katalog + jeden `..%2f` więcej**: `/resources/wcd/..%2f..%2fmy-account` rozwija się u origina do `/my-account` (wcd kasuje 1. `..`, resources kasuje 2.), a cache widzi świeży string zaczynający się od `/resources/`. Każdy dodatkowy śmieciowy segment = jeden dodatkowy `..%2f`.

---

## 💥 Moje potknięcia

- **Reguła katalogu vs reguła rozszerzenia** — `tracking.js` jest JEDNOCZEŚNIE pod `/resources` i kończy się `.js`; z jednego requestu nie wiesz, która reguła go złapała. Nasz payload nie ma `.js` → potrzebna reguła **katalogu** → potwierdzasz Krokiem 3 (dry run).
- **Nie odpalaj payloadu na sobie** (albo użyj świeżej ścieżki) — dry run cachuje TWOJE konto na ~30 s; jak potem wyślesz carlosowi ten sam URL, dostanie Twoją kopię. Ja w ogóle pominąłem dry run na sobie i od razu dałem świeżą ścieżkę dla carlosa.
- Slash **musi** być `%2f` (patrz wyżej); sprawdź, że traversal wraca na Twoje konto, nie na login.

---

## Czego ten lab dokłada

Trzeci wymiar rozbieżności: **stopień normalizacji ścieżki** (dekoduje/rozwija vs bierze dosłownie). Wprowadza encoded slash `%2f` jako klin i technikę świeżego klucza przez dodatkowy `..%2f`. Pytanie **„dekoduje czy bierze dosłownie?"** wraca w SSRF, path traversal, WAF bypassach. Następny [[04-cache-normalization]] to **lustro**: cache normalizuje, origin nie.
