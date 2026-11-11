# Kontrakt API

- **Wersja:** 0.1 (szkic, część 1: zasady ogólne)
- **Data:** 2026-11-11
- **Powiązane:** [wymagania](wymagania.md) (v0.2), [model domeny](model-domeny.md) (v0.1), [ADR-0003](adr/0003-uwierzytelnianie-klienta-mobilnego.md)

Dokument opisuje API REST, z którego korzysta aplikacja mobilna. Składa się z dwóch części:

- **zasady ogólne** (sekcje 1–7): reguły, które obowiązują każdy endpoint,
- **tabela endpointów** (sekcja 8): metoda, adres, cel, statusy i uprawnienia każdego endpointu.

Szczegółowy kształt żądań i odpowiedzi (pola, typy, ograniczenia) opisuje **wyłącznie** specyfikacja OpenAPI generowana z kodu (N-16), dostępna w uruchomionej aplikacji przez Swagger UI. Ten dokument tego nie powtarza, żeby nic nie było opisane w dwóch miejscach.

Dokument jest żywy: zmienia się razem z kodem, w tym samym Pull Requeście.

## 1. Zasady ogólne

- Komunikacja wyłącznie przez HTTPS (N-03).
- Wszystkie endpointy mają prefiks `/api`.
- Treść żądań i odpowiedzi to JSON w UTF-8 (`Content-Type: application/json`). Wyjątek: błędy (sekcja 4).
- Nazwy pól w JSON-ie w stylu `camelCase` (np. `firstName`). Wartości wyliczeniowe wielkimi literami z podkreśleniem (np. `MONDAY`, `WEIGHT_LOSS`).
- Teksty dla użytkownika (`title`, `detail` w błędach) są po polsku (N-19). Jednostki: kilogramy i centymetry.

## 2. Adresy i zasoby

- Adres wskazuje zasób, a metoda HTTP określa czynność. Nazwy zasobów to rzeczowniki w liczbie mnogiej, po angielsku, małymi literami z myślnikami (np. `/api/clients`, `/api/personal-records`).
- Zagnieżdżamy najwyżej jeden poziom: `/api/clients/{id}/plans`. Zasób z własnym identyfikatorem jest też dostępny bezpośrednio: `/api/plans/{id}`.
- **Plan jest zapisywany i odczytywany w całości:** razem z dniami treningowymi, ćwiczeniami w dniach i seriami planowanymi. Dni, ćwiczenia w dniu i serie nie mają osobnych endpointów. Dzięki temu serwer sprawdza plan jako całość i zapisuje go w jednej transakcji (R-08).
- **Dane zalogowanego podopiecznego są dostępne pod `/api/me/…`** (np. `/api/me/measurements`). Tożsamość wynika z tokenu, więc w adresie nie ma identyfikatora, który dałoby się podmienić.

**Metody i statusy powodzenia:**

| Metoda | Użycie | Status |
|---|---|---|
| `GET` | odczyt | `200 OK` |
| `POST` | utworzenie zasobu | `201 Created` z nagłówkiem `Location` |
| `POST` | czynność, która nie tworzy zasobu (np. logowanie) | `200 OK` albo `204 No Content` |
| `PUT` | zastąpienie całego zasobu | `200 OK` |
| `DELETE` | usunięcie | `204 No Content` |

## 3. Uwierzytelnianie i dostęp

- Każde żądanie poza logowaniem, resetem hasła i przyjęciem zaproszenia wymaga nagłówka `Authorization: Bearer <access token>` (ADR-0003).
- **401 Unauthorized:** brak tokenu, token nieprawidłowy albo wygasły.
- **403 Forbidden:** rola nie ma prawa do danej czynności, niezależnie od tego, o który zasób chodzi (np. podopieczny wywołuje `POST /api/plans`).
- **404 Not Found:** zasób nie istnieje **albo należy do kogoś innego**. Obie sytuacje dają identyczną odpowiedź, żeby nie zdradzać, czy cudze dane istnieją (wymagania, sekcja 6).

## 4. Błędy

Wszystkie błędy, także 401 i 403, mają format **Problem Details** ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) i `Content-Type: application/problem+json`.

**Pola standardowe:** `type`, `title`, `status`, `detail`, `instance`.

**Pole `type`** identyfikuje rodzaj błędu i to po nim klient rozpoznaje problem, nigdy po tekście w `detail`:

- błąd, który nie znaczy nic ponad status HTTP (401, 403, 404, 500): `type` = `about:blank`, a `title` to standardowa nazwa statusu,
- błąd, na który klient reaguje w określony sposób: własny, pełny adres `https://<domena>/problems/<nazwa>` (domena: A-01).

Raz nadanego `type` nigdy nie zmieniamy (sekcja 6).

**Zarejestrowane typy błędów:**

| `type` (`<nazwa>`) | Status | Kiedy |
|---|---|---|
| `validation-error` | 400 | dane w żądaniu nie przechodzą walidacji |

Kolejne typy dopisujemy razem z endpointami, które ich używają.

**Błędy walidacji** zawierają rozszerzenie `errors`: listę **wszystkich** błędów, nie tylko pierwszego. Każdy wpis ma:

- `pointer`: miejsce błędu w treści żądania jako JSON Pointer ([RFC 6901](https://www.rfc-editor.org/rfc/rfc6901)) w postaci fragmentu URI, np. `#/days/0/exercises/2/sets/1/reps`. Błąd dotyczący kilku pól wskazuje najbliższego wspólnego rodzica (np. `#/days`),
- `detail`: opis błędu dla użytkownika, po polsku.

```json
{
  "type": "https://<domena>/problems/validation-error",
  "title": "Nieprawidłowe dane",
  "status": 400,
  "detail": "Plan zawiera błędy.",
  "errors": [
    { "pointer": "#/name", "detail": "Nazwa jest wymagana." },
    { "pointer": "#/days/0/exercises/2/sets/1/reps", "detail": "Liczba powtórzeń musi być większa od zera." }
  ]
}
```

**Błąd serwera (500)** nie zawiera szczegółów technicznych (stack trace, nazwy tabel, komunikaty bazy). Szczegóły trafiają wyłącznie do logów (N-06).

## 5. Listy: paginacja, sortowanie, filtrowanie

- Listy są stronicowane parametrami `page` (od 0) i `size`. Domyślnie `size=20`, maksymalnie `100`. Większa wartość kończy się błędem 400.
- Sortowanie parametrem `sort=pole,asc|desc`. Dla każdej listy dozwolone pola sortowania są wymienione w tabeli endpointów. Inne pole kończy się błędem 400. Serwer zawsze dokłada na końcu sortowanie po identyfikatorze, żeby kolejność była stabilna między stronami.
- Filtry to parametry zapytania nazwane jak pola (np. `?status=ARCHIVED`). Wyszukiwanie tekstowe: parametr `q`, bez rozróżniania wielkości liter.
- Odpowiedź z listą ma stały kształt:

```json
{
  "content": [ ],
  "page": { "size": 20, "number": 0, "totalElements": 87, "totalPages": 5 }
}
```

**Ograniczenie:** paginacja oparta na stronach może pokazać ten sam element dwa razy, jeśli w trakcie przewijania na początku listy pojawi się nowy element. Przy skali MVP to akceptowalne. Klient usuwa duplikaty po identyfikatorze. Gdy stanie się to problemem, można dodać paginację kursorem jako nowe, opcjonalne parametry.

## 6. Zgodność wsteczna i zmiany API

API nie ma wersji w adresie. Spełniamy N-17 (zgodność z aplikacjami z ostatnich 3 miesięcy) tak, że wprowadzamy wyłącznie zmiany zgodne wstecz.

**Zmiany dozwolone:**
- nowy endpoint,
- nowe pole w odpowiedzi,
- nowy **opcjonalny** parametr albo pole w żądaniu,
- nowy typ błędu.

**Zmiany zabronione:**
- usunięcie albo zmiana nazwy pola, parametru lub endpointu,
- zmiana typu albo znaczenia pola,
- nowe **wymagane** pole lub parametr w żądaniu, zaostrzenie walidacji istniejącego pola,
- zmiana `type` istniejącego błędu albo statusu odpowiedzi.

**Gdy zmiana niezgodna jest konieczna:** nowa wersja powstaje obok starej (nowy endpoint albo nowe pole). Stara zostaje oznaczona w OpenAPI jako `deprecated` i jest usuwana najwcześniej po 3 miesiącach.

**Obowiązek klienta:** aplikacja mobilna ignoruje pola, których nie zna. Bez tego nawet dodanie pola do odpowiedzi byłoby zmianą niezgodną.

## 7. Daty i czas

| Rodzaj | Przykłady | Format w JSON-ie | Typ w kodzie / bazie |
|---|---|---|---|
| Moment w czasie | zapis treningu, wysłanie i wygaśnięcie zaproszenia | `"2026-11-11T07:03:00Z"` (RFC 3339, zawsze UTC) | `Instant` / `timestamptz` |
| Data kalendarzowa | data pomiaru, data rekordu, początek planu | `"2026-11-11"` | `LocalDate` / `date` |
| Czas trwania | przerwa po serii, czas serii | liczba całkowita sekund | `int` |
| Dzień tygodnia | sugerowane dni treningowe | `MONDAY` … `SUNDAY` | `DayOfWeek` |

- Momenty w czasie są przechowywane i przesyłane w UTC, a w strefie Europe/Warsaw wyświetla je aplikacja (N-18).
- **„Dzisiaj” serwer wyznacza w strefie Europe/Warsaw,** a nie w strefie serwera. Dotyczy to m.in. domyślnej daty pomiaru (W-14), zakazu dat z przyszłości (R-15), statusów podopiecznych (W-04) i bieżącego tygodnia (W-12). Strefa jest ustawieniem konfiguracyjnym.

## 8. Endpointy

*Do uzupełnienia w wersji 0.2.* Tabela endpointów ekran po ekranie, z rozstrzygnięciem otwartych kwestii M-01 i M-02 z modelu domeny.

## 9. Otwarte kwestie

| Nr | Kwestia | Rozstrzygnąć przy |
|---|---|---|
| A-01 | Domena API, potrzebna do adresów `type` w błędach | przed pierwszym wdrożeniem |
| A-02 | Typ identyfikatorów zasobów: kolejne liczby czy UUID | tabeli endpointów (sekcja 8) |

## Historia zmian

| Wersja | Data | Zmiany |
|---|---|---|
| 0.1 | 2026-11-11 | Zasady ogólne: adresy, dostęp, błędy, listy, zgodność wsteczna, daty |
