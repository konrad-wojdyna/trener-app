# Kontrakt API

- **Wersja:** 0.2 (szkic)
- **Data:** 2026-11-12
- **Powiązane:** [wymagania](wymagania.md) (v0.3), [model domeny](model-domeny.md) (v0.2), [ADR-0003](adr/0003-uwierzytelnianie-klienta-mobilnego.md)

Dokument opisuje API REST, z którego korzysta aplikacja mobilna. Składa się z dwóch części:

- **zasady ogólne** (sekcje 1–7): reguły, które obowiązują każdy endpoint,
- **tabela endpointów** (sekcja 8): metoda, adres, cel, statusy i uprawnienia każdego endpointu.

Szczegółowy kształt żądań i odpowiedzi (pola, typy, ograniczenia) opisuje **wyłącznie** specyfikacja OpenAPI generowana z kodu (N-16), dostępna w uruchomionej aplikacji przez Swagger UI. Ten dokument tego nie powtarza, żeby nic nie było opisane w dwóch miejscach.

Dokument jest żywy: zmienia się razem z kodem, w tym samym Pull Requeście.

## 1. Zasady ogólne

- Komunikacja wyłącznie przez HTTPS (N-03).
- Wszystkie endpointy mają prefiks `/api`.
- Treść żądań i odpowiedzi to JSON w UTF-8 (`Content-Type: application/json`). Wyjątek: błędy (sekcja 4).
- Nazwy pól w JSON-ie w stylu `camelCase` (np. `firstName`). Wartości wyliczeniowe wielkimi literami z podkreśleniem (np. `MONDAY`, `WEIGHT_LOSS`). API przesyła kody, a polskie nazwy pokazuje aplikacja.
- Teksty dla użytkownika (`title`, `detail` w błędach) są po polsku (N-19). Jednostki: kilogramy i centymetry.

## 2. Adresy i zasoby

- Adres wskazuje zasób, a metoda HTTP określa czynność. Nazwy zasobów to rzeczowniki w liczbie mnogiej, po angielsku, małymi literami z myślnikami (np. `/api/clients`, `/api/personal-records`).
- Zagnieżdżamy najwyżej jeden poziom: `/api/clients/{id}/workouts`. Zasób z własnym identyfikatorem jest też dostępny bezpośrednio: `/api/workouts/{id}`.
- Wartości wyliczane z kolekcji lub zasobu mają adres po ukośniku (np. `/api/clients/counts`, `/api/clients/{id}/summary`, `/api/me/personal-records/best`). Czynności, które nie są zapisem ani usunięciem zasobu, też mają adres po ukośniku i metodę `POST` (np. `/api/clients/{id}/archive`).
- **Identyfikatory zasobów to UUID w wersji 7** ([RFC 9562](https://www.rfc-editor.org/rfc/rfc9562)), przesyłane w JSON-ie jako tekst. Identyfikator nadaje serwer, z wyjątkiem treningu, któremu identyfikator nadaje aplikacja (sekcja 8.8).
- **Plan jest zapisywany i odczytywany w całości:** razem z dniami treningowymi, ćwiczeniami w dniach i seriami planowanymi. Dni, ćwiczenia w dniu i serie nie mają osobnych endpointów. Dzięki temu serwer sprawdza plan jako całość i zapisuje go w jednej transakcji (R-08).
- **Dane zalogowanego podopiecznego są dostępne pod `/api/me/…`** (np. `/api/me/measurements`). Tożsamość wynika z tokenu, więc w adresie nie ma identyfikatora, który dałoby się podmienić. Wszystko pod `/api/me/…` jest wyłącznie dla podopiecznego; trener dostaje 403.

**Wybór metody:** gdy adres zasobu zna klient (np. pomiar z datą w adresie), zapis to `PUT` pod tym adresem. Gdy adres nadaje serwer, zapis to `POST` na kolekcję. Wyjątek: trening ma identyfikator nadany przez aplikację, ale używa `POST`, bo zapisanego treningu nie wolno zastąpić (R-11), a `PUT` oznacza zastąpienie.

**Metody i statusy powodzenia:**

| Metoda | Użycie | Status |
|---|---|---|
| `GET` | odczyt | `200 OK` |
| `POST` | utworzenie zasobu | `201 Created`; z nagłówkiem `Location`, gdy utworzony zasób ma własny adres |
| `POST` | czynność, która nie tworzy zasobu (np. logowanie, archiwizacja) | `200 OK` albo `204 No Content` |
| `PUT` | utworzenie zasobu pod adresem znanym klientowi | `201 Created` |
| `PUT` | zastąpienie istniejącego zasobu | `200 OK` |
| `DELETE` | usunięcie | `204 No Content` |

**Ponowne wysłanie zapisu z identyfikatorem nadanym przez klienta** (trening): pierwsze wysłanie zwraca `201 Created`, a każde kolejne z tym samym identyfikatorem `200 OK` z istniejącym zasobem. Serwer nie porównuje treści powtórzonego żądania.

**Ograniczenie:** API nie stosuje blokowania optymistycznego. Gdy dwie osoby jednocześnie zapisują ten sam zasób (np. plan), wygrywa ostatni zapis. Przy jednym trenerze to akceptowalne. Później można dodać opcjonalny nagłówek `If-Match` ze statusem `412 Precondition Failed`, co jest zmianą zgodną wstecz.

## 3. Uwierzytelnianie i dostęp

- Endpointy `/api/auth/*` (`login`, `refresh`, `logout`, `forgot-password`, `reset-password`, `accept-invitation`) są publiczne. `refresh` i `logout` przyjmują refresh token w treści żądania, bez access tokenu.
- Każde inne żądanie wymaga nagłówka `Authorization: Bearer <access token>` (ADR-0003).
- **401 Unauthorized:** brak tokenu, token nieprawidłowy albo wygasły. Przy logowaniu: błędne dane logowania. Błędny e-mail, błędne hasło oraz konto zablokowane lub zarchiwizowane dają identyczną odpowiedź.
- **403 Forbidden:** rola nie ma prawa do danej czynności, niezależnie od tego, o który zasób chodzi (np. podopieczny wywołuje `POST /api/plans`), **albo** zasób jest widoczny, ale tylko do odczytu (np. edycja ćwiczenia wspólnego).
- **404 Not Found:** zasób nie istnieje **albo należy do kogoś innego**. Obie sytuacje dają identyczną odpowiedź, żeby nie zdradzać, czy cudze dane istnieją (wymagania, sekcja 6).
- **429 Too Many Requests:** przekroczony limit prób (N-05), z nagłówkiem `Retry-After`.

## 4. Błędy

Wszystkie błędy, także 401 i 403, mają format **Problem Details** ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) i `Content-Type: application/problem+json`.

**Pola standardowe:** `type`, `title`, `status`, `detail`, `instance`.

**Pole `type`** identyfikuje rodzaj błędu i to po nim klient rozpoznaje problem, nigdy po tekście w `detail`:

- błąd, który nie znaczy nic ponad status HTTP w danym endpoincie: `type` = `about:blank`, a `title` to standardowa nazwa statusu,
- błąd, na który klient reaguje w określony sposób, w tym ten sam status zwracany przez jeden endpoint z różnych przyczyn, na które klient reaguje inaczej: własny, pełny adres `https://<domena>/problems/<nazwa>` (domena: A-01).

Raz nadanego `type` nigdy nie zmieniamy (sekcja 6).

**Zarejestrowane typy błędów:**

| `type` (`<nazwa>`) | Status | Kiedy |
|---|---|---|
| `validation-error` | 400 | dane w żądaniu nie przechodzą walidacji |
| `invalid-link` | 400 | token z linku w mailu (reset hasła, zaproszenie) jest nieprawidłowy, wygasł albo został już wykorzystany |

Kolejne typy dopisujemy razem z endpointami, które ich używają.

**Błędne odwołanie w treści żądania** (np. nieistniejący `clientId` albo `exerciseId`) to `validation-error` ze wskazaniem pola, a nie 404. Status 404 dotyczy wyłącznie zasobu wskazanego w adresie.

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
- Filtry to parametry zapytania nazwane jak pola (np. `?archived=true`). Wyszukiwanie tekstowe: parametr `q`, bez rozróżniania wielkości liter.
- Odpowiedź z listą ma stały kształt:

```json
{
  "content": [ ],
  "page": { "size": 20, "number": 0, "totalElements": 87, "totalPages": 5 }
}
```

**Wyjątek: serie czasowe do wykresów** (pomiary) nie są stronicowane. Zamiast stron mają zakres dat w parametrach `from` i `to`, najwyżej 366 dni. Przy jednym wpisie dziennie odpowiedź ma więc ograniczoną wielkość, a wykres dostaje wszystkie punkty naraz.

**Ograniczenie:** paginacja oparta na stronach może pokazać ten sam element dwa razy, jeśli w trakcie przewijania na początku listy pojawi się nowy element. Przy skali MVP to akceptowalne. Klient usuwa duplikaty po identyfikatorze. Gdy stanie się to problemem, można dodać paginację kursorem jako nowe, opcjonalne parametry.

## 6. Zgodność wsteczna i zmiany API

API nie ma wersji w adresie. Spełniamy N-17 (zgodność z aplikacjami z ostatnich 3 miesięcy) tak, że wprowadzamy wyłącznie zmiany zgodne wstecz.

**Zmiany dozwolone:**
- nowy endpoint,
- nowe pole w odpowiedzi,
- nowa wartość w liście wyliczeniowej w odpowiedzi (np. nowa partia mięśniowa),
- nowy **opcjonalny** parametr albo pole w żądaniu,
- nowy typ błędu.

**Zmiany zabronione:**
- usunięcie albo zmiana nazwy pola, parametru lub endpointu,
- zmiana typu albo znaczenia pola,
- nowe **wymagane** pole lub parametr w żądaniu, zaostrzenie walidacji istniejącego pola,
- zmiana `type` istniejącego błędu albo statusu odpowiedzi.

**Gdy zmiana niezgodna jest konieczna:** nowa wersja powstaje obok starej (nowy endpoint albo nowe pole). Stara zostaje oznaczona w OpenAPI jako `deprecated` i jest usuwana najwcześniej po 3 miesiącach.

**Obowiązek klienta:** aplikacja mobilna ignoruje pola, których nie zna, a nieznana wartość z listy wyliczeniowej nie może spowodować błędu aplikacji (aplikacja pokazuje wtedy wartość zastępczą). Bez tego nawet dodanie pola do odpowiedzi byłoby zmianą niezgodną.

## 7. Daty i czas

| Rodzaj | Przykłady | Format w JSON-ie | Typ w kodzie / bazie |
|---|---|---|---|
| Moment w czasie | rozpoczęcie i zakończenie treningu, wysłanie i wygaśnięcie zaproszenia | `"2026-11-11T07:03:00Z"` (RFC 3339, zawsze UTC) | `Instant` / `timestamptz` |
| Data kalendarzowa | data pomiaru, data rekordu, początek planu | `"2026-11-11"` | `LocalDate` / `date` |
| Czas trwania | przerwa po serii, czas serii | liczba całkowita sekund | `int` |
| Dzień tygodnia | sugerowane dni treningowe | `MONDAY` … `SUNDAY` | `DayOfWeek` |

- Momenty w czasie są przechowywane i przesyłane w UTC, a w strefie Europe/Warsaw wyświetla je aplikacja (N-18).
- **„Dzisiaj” serwer wyznacza w strefie Europe/Warsaw,** a nie w strefie serwera. Dotyczy to m.in. domyślnej daty pomiaru (W-14), zakazu dat z przyszłości (R-15), statusów podopiecznych (W-04), bieżącego tygodnia (W-12, od poniedziałku do niedzieli) i dnia, do którego liczy się trening. Strefa jest ustawieniem konfiguracyjnym.

## 8. Endpointy

Wszystkie endpointy poza `/api/auth/*` mogą zwrócić także 401 (sekcja 3); tabele tego nie powtarzają. W kolumnie „Błędy” nazwa w nawiasie przy 400 to `type` błędu (sekcja 4); przy innych statusach to przyczyna, a `type` = `about:blank`.

### 8.1 Logowanie (ekran 1; W-01, W-02, W-03, W-06)

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `POST` | `/api/auth/login` | Logowanie e-mailem i hasłem; zwraca tokeny i rolę | 200 | 400, 401, 429 | publiczny |
| `POST` | `/api/auth/refresh` | Nowa para tokenów za refresh token; zwraca też rolę | 200 | 400, 401 | publiczny (refresh token w treści) |
| `POST` | `/api/auth/logout` | Unieważnia refresh token z tego urządzenia | 204 | 400 | publiczny (refresh token w treści) |
| `POST` | `/api/auth/forgot-password` | Wysyła link do resetu, jeśli konto istnieje | 204 | 400, 429 | publiczny |
| `POST` | `/api/auth/reset-password` | Nowe hasło na podstawie tokenu z linku | 204 | 400 (`validation-error`, `invalid-link`) | publiczny |
| `POST` | `/api/auth/accept-invitation` | Hasło na podstawie tokenu z zaproszenia, aktywacja konta | 204 | 400 (`validation-error`, `invalid-link`) | publiczny |

- `forgot-password` zwraca 204 także wtedy, gdy konto nie istnieje.
- Limit prób (N-05) liczony jest na adres e-mail, także nieistniejący. Ten sam limit obowiązuje `forgot-password`.
- Tokeny z linków w mailach są przesyłane w treści żądania, nigdy w adresie (N-06). Link z maila prowadzi na stronę web z formularzem, a token jest we fragmencie adresu (`#token=…`), więc nie trafia do logów serwera.
- `reset-password` i `accept-invitation` nie logują użytkownika. Po ustawieniu hasła użytkownik loguje się przez `login`.
- Czas życia, rotację i unieważnianie tokenów opisuje osobny ADR.

### 8.2 Lista podopiecznych (ekran 2; W-04)

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `GET` | `/api/clients` | Lista podopiecznych z aktywnym kontem. Filtry: `archived` (domyślnie `false`), `needsAttention`; `q` (imię, nazwisko, e-mail). Sortowanie: `name` (imię, nazwisko; domyślnie rosnąco) | 200 | 400, 403 | trener |
| `GET` | `/api/clients/counts` | Liczba podopiecznych aktywnych, wymagających uwagi i zarchiwizowanych | 200 | 403 | trener |

- `status` (`ON_PLAN`, `NO_RECENT_WORKOUT`, `PLAN_ENDING`, `NO_PLAN`) i `needsAttention` wylicza serwer. `PLAN_ENDING` obejmuje także plan po planowanej dacie końca (R-17).
- Zaproszeni bez konta nie są na liście ani w liczbach (sekcja 8.6, `GET /api/invitations`).

### 8.3 Profil podopiecznego (ekran 3; W-05, W-07, N-08)

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `GET` | `/api/clients/{id}` | Dane profilu i notatka trenera | 200 | 403, 404 | trener |
| `GET` | `/api/clients/{id}/summary` | Postęp aktywnego planu, waga i obwód pasa (ostatni pomiar i zmiana od pierwszego), frekwencja z 4 tygodni | 200 | 403, 404 | trener |
| `GET` | `/api/clients/{id}/workouts` | Historia treningów, stronicowana. Sortowanie: `startedAt` (domyślnie malejąco) | 200 | 400, 403, 404 | trener |
| `GET` | `/api/workouts/{id}` | Szczegóły treningu: serie (cel i wynik), uwagi podopiecznego | 200 | 403, 404 | trener |
| `PUT` | `/api/clients/{id}/note` | Notatka trenera (do 2000 znaków; pusta usuwa notatkę) | 200 | 400, 403, 404 | trener |
| `POST` | `/api/clients/{id}/archive` | Archiwizacja; odcina dostęp i unieważnia refresh tokeny podopiecznego | 204 | 403, 404, 409 (zaproszony bez konta) | trener |
| `POST` | `/api/clients/{id}/unarchive` | Przywrócenie z archiwum | 204 | 403, 404 | trener |
| `DELETE` | `/api/clients/{id}` | Trwałe usunięcie podopiecznego, jego konta i wszystkich danych | 204 | 403, 404, 409 (aktywny, niezarchiwizowany) | trener |

- `summary`: brak danych (planu, pomiarów, treningów) to stan normalny, a odpowiednie pola mają wartość `null`.
- `archive` i `unarchive` są idempotentne: ponowne wywołanie dla podopiecznego w docelowym stanie zwraca 204.
- `DELETE` jest dozwolony dla podopiecznego zarchiwizowanego albo zaproszonego, który nie aktywował konta (R-18).
- API nie przewiduje edycji danych podopiecznego.

### 8.4 Kreator planu (ekran 4; W-10, W-11)

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `POST` | `/api/plans` | Nowy plan w całości (`clientId` w treści); zapis aktywuje plan i kończy poprzedni (R-07) | 201 + `Location` | 400, 403 | trener |
| `GET` | `/api/plans/{id}` | Cały plan do edycji | 200 | 403, 404 | trener |
| `PUT` | `/api/plans/{id}` | Zastąpienie planu w całości (W-11) | 200 | 400, 403, 404 | trener |

- `clientId` musi wskazywać podopiecznego z aktywnym kontem (inaczej `validation-error`).
- Dzień treningowy ma opcjonalne `estimatedMinutes` (10–240). Plan ma opcjonalne `suggestedDays`: lista `MONDAY` … `SUNDAY`.
- `PUT`: dni treningowe zachowują identyfikatory. Dzień z `id` jest aktualizowany, dzień bez `id` jest nowy, a dzień nieobecny w żądaniu jest usuwany z planu (R-16). Ćwiczenia w dniu i serie są zastępowane w całości.
- Ćwiczenie ukryte może zostać w planie, który już je zawiera. Dodanie ukrytego ćwiczenia do dnia, w którym go nie było, kończy się `validation-error`.
- Przerwa jest podawana dla każdej serii osobno.

### 8.5 Biblioteka ćwiczeń (ekran 5; W-08, W-09)

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `GET` | `/api/exercises` | Lista bez ukrytych; `q` (nazwa), filtry `muscleGroup`, `custom`. Sortowanie: `name` | 200 | 400, 403 | trener |
| `GET` | `/api/exercises/{id}` | Szczegóły ćwiczenia, także ukrytego | 200 | 403, 404 | trener |
| `POST` | `/api/exercises` | Nowe własne ćwiczenie | 201 + `Location` | 400, 403, 409 (nazwa zajęta) | trener |
| `PUT` | `/api/exercises/{id}` | Edycja własnego ćwiczenia | 200 | 400, 403 (ćwiczenie wspólne), 404, 409 (nazwa zajęta) | trener |
| `POST` | `/api/exercises/{id}/hide` | Ukrycie z biblioteki (R-06) | 204 | 403 (ćwiczenie wspólne), 404 | trener |
| `DELETE` | `/api/exercises/{id}` | Usunięcie własnego, nigdzie nieużytego ćwiczenia | 204 | 403 (ćwiczenie wspólne), 404, 409 (ćwiczenie użyte) | trener |

- `muscleGroup`: `CHEST`, `BACK`, `SHOULDERS`, `ARMS`, `LEGS`, `GLUTES`, `CORE`, `FULL_BODY`.
- `equipment`: `BARBELL`, `DUMBBELL`, `MACHINE`, `CABLE`, `KETTLEBELL`, `BAND`, `BODYWEIGHT`, `OTHER`.
- Nazwa jest unikalna wśród wszystkich widocznych ćwiczeń, wspólnych i własnych, bez rozróżniania wielkości liter (R-05).
- Na 409 przy `DELETE` aplikacja proponuje ukrycie ćwiczenia.

### 8.6 Zaproszenie podopiecznego (ekran 6; W-06)

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `POST` | `/api/clients` | Nowy podopieczny (`firstName`, `lastName`, `email`, opcjonalnie `goal`), pierwsze zaproszenie i mail | 201 + `Location` | 400, 403, 409 (e-mail zajęty) | trener |
| `GET` | `/api/invitations` | Podopieczni bez konta z ostatnim zaproszeniem (data wysłania, data wygaśnięcia, status `PENDING` / `EXPIRED`). Sortowanie: `sentAt` (domyślnie malejąco) | 200 | 400, 403 | trener |
| `POST` | `/api/clients/{id}/invitations` | Nowe zaproszenie i mail; poprzednie zaproszenie zostaje unieważnione (R-03) | 201 | 403, 404, 409 (podopieczny ma już konto) | trener |

- E-mail jest zajęty, gdy należy do dowolnego podopiecznego (aktywnego, zarchiwizowanego lub zaproszonego) albo do konta trenera (R-01).
- `POST /api/clients/{id}/invitations` nie zwraca `Location`, bo zaproszenie nie ma własnego adresu.

### 8.7 Dziś (ekran 7; W-12)

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `GET` | `/api/me` | Dane zalogowanego podopiecznego: e-mail, imię, nazwisko, cel | 200 | 403 | podopieczny |
| `GET` | `/api/me/today` | Dane ekranu „Dziś”: dzisiejsza data; bieżący tydzień (dni z treningiem, dni sugerowane); plan (nazwa, `currentWeek`, liczba tygodni, `endsOn`); następny trening w całości (nazwa dnia, `estimatedMinutes`, ćwiczenia z filmem, serie z celem i przerwą, podpowiedź „ostatni ciężar”); `measurementDue` | 200 | 403 | podopieczny |

- `/api/me` nie zwraca notatki trenera.
- Brak aktywnego planu to stan normalny: `plan` i `nextWorkout` mają wartość `null`.
- `measurementDue` = `true`, gdy ostatni pomiar jest starszy niż 7 dni albo nie ma żadnego.
- Po planowanej dacie końca `currentWeek` może być większy niż liczba tygodni planu (R-17).
- Trening liczy się do dnia, w którym go rozpoczęto (`startedAt` w strefie Europe/Warsaw).

### 8.8 Trening w trakcie (ekran 8; W-13)

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `POST` | `/api/me/workouts` | Zapis całego treningu naraz (R-12): `id` nadane przez aplikację, `trainingDayId`, `startedAt`, `finishedAt`, `status` (`COMPLETED` / `PARTIAL`), ćwiczenia (pominięte, uwaga do 500 znaków) z seriami (kopia celu, wynik, czy wykonana) | 201 + `Location`; powtórzenie z tym samym `id` → 200 | 400, 403, 409 (`id` zajęty przez trening innego podopiecznego) | podopieczny |

- Dane do wyświetlenia aplikacja bierze z `GET /api/me/today`. Odrzucony trening nie jest wysyłany (R-13).
- `id` to UUID w wersji 7, generowany przez aplikację przy rozpoczęciu treningu. Odpowiedź 200 na powtórzenie oznacza, że trening jest zapisany dokładnie raz (sekcja 2).
- Kopię celu serii (R-11) przysyła aplikacja: to cel, który podopieczny widział w chwili treningu.
- `trainingDayId` musi należeć do planu tego podopiecznego, także zakończonego, i może wskazywać dzień usunięty z planu (R-16); inaczej `validation-error`. Trening do dnia ze starszego planu nie przesuwa kolejki aktywnego planu (R-09).
- `finishedAt` ≥ `startedAt`, a żaden z nich nie może być z przyszłości (tolerancja 5 minut na różnicę zegarów).
- `status` wybiera użytkownik („Zakończ” albo „Przerwij → zapisz częściowo”); serwer go nie wylicza.
- `Location` wskazuje `/api/workouts/{id}`. Podopieczny w MVP nie odczytuje zapisanych treningów (wymagania, sekcja 6).

### 8.9 Postępy (ekran 9; W-14, W-15)

**Podopieczny:**

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `GET` | `/api/me/measurements` | Pomiary z zakresu dat, rosnąco po dacie; `from`, `to` (domyślnie ostatnie 8 tygodni) | 200 | 400, 403 | podopieczny |
| `PUT` | `/api/me/measurements/{date}` | Zapis pomiaru z danego dnia; istniejący jest nadpisywany (R-14) | 201 (nowy) / 200 (nadpisany) | 400, 403 | podopieczny |
| `DELETE` | `/api/me/measurements/{date}` | Usunięcie pomiaru | 204 | 403, 404 | podopieczny |
| `GET` | `/api/me/personal-records/best` | Aktualny rekord (ciężar i data) dla każdego ćwiczenia, w którym jest wpis | 200 | 403 | podopieczny |
| `GET` | `/api/me/personal-records` | Historia wpisów, stronicowana; filtr `exerciseId`. Sortowanie: `date` (domyślnie malejąco) | 200 | 400, 403 | podopieczny |
| `POST` | `/api/me/personal-records` | Nowy wpis: `exerciseId`, ciężar 1–500 kg, data (domyślnie dziś, nie z przyszłości) | 201 + `Location` | 400, 403 | podopieczny |
| `DELETE` | `/api/me/personal-records/{id}` | Usunięcie błędnego wpisu | 204 | 403, 404 | podopieczny |
| `GET` | `/api/me/exercises` | Ćwiczenia ze wszystkich własnych planów (aktywnego i zakończonych), do wyboru przy rekordzie; `q`. Sortowanie: `name` | 200 | 400, 403 | podopieczny |

**Trener (tylko odczyt, te same kształty odpowiedzi):**

| Metoda | Adres | Cel | Sukces | Błędy | Dostęp |
|---|---|---|---|---|---|
| `GET` | `/api/clients/{id}/measurements` | Jak `GET /api/me/measurements` | 200 | 400, 403, 404 | trener |
| `GET` | `/api/clients/{id}/personal-records/best` | Jak `GET /api/me/personal-records/best` | 200 | 403, 404 | trener |
| `GET` | `/api/clients/{id}/personal-records` | Jak `GET /api/me/personal-records` | 200 | 400, 403, 404 | trener |

- Pomiary nie są stronicowane; zakres `from`–`to` ma najwyżej 366 dni (sekcja 5).
- Data pomiaru nie może być z przyszłości (R-15).
- Każdy rekord jest osobnym wpisem w historii; wpisy nie są nadpisywane.
- Aktualny rekord to największy ciężar; przy remisie liczy się najwcześniejsza data.
- `exerciseId` spoza własnych planów kończy się `validation-error`. Ćwiczenie ukryte, ale obecne w planie, jest dozwolone.

## 9. Otwarte kwestie

| Nr | Kwestia | Status |
|---|---|---|
| A-01 | Domena API, potrzebna do adresów `type` w błędach | Otwarte; rozstrzygnąć przed pierwszym wdrożeniem |
| A-02 | Typ identyfikatorów zasobów: kolejne liczby czy UUID | **Rozstrzygnięte (2026-11-12):** UUID w wersji 7 dla wszystkich zasobów (sekcja 2). Trening i tak ma identyfikator nadany przez aplikację, więc w API jest jeden rodzaj identyfikatora; kolejne liczby zdradzają liczbę rekordów i dają się zgadywać; wersja 7 zaczyna się od czasu, więc nowe wpisy trafiają na koniec indeksu. Koszt: 16 bajtów zamiast 8 |
| A-03 | Gdzie działa strona web z formularzami „ustaw hasło” i „nowe hasło”, otwierana z linku w mailu | Otwarte; rozstrzygnąć przy budowie W-06 |

## Historia zmian

| Wersja | Data | Zmiany |
|---|---|---|
| 0.1 | 2026-11-11 | Zasady ogólne: adresy, dostęp, błędy, listy, zgodność wsteczna, daty |
| 0.2 | 2026-11-12 | Tabela endpointów dla ekranów 1–9 (sekcja 8). Zasady ogólne: identyfikatory UUIDv7, wybór metody `PUT` / `POST`, `PUT` tworzący zasób (201), idempotentny zapis z identyfikatorem klienta, wartości wyliczane i czynności w adresach, `/api/me/…` tylko dla podopiecznego, brak blokowania optymistycznego (sekcja 2); publiczne endpointy `/api/auth/*`, 401 przy logowaniu, 403 dla zasobów tylko do odczytu, 429 (sekcja 3); typ `invalid-link`, błędne odwołanie w treści jako `validation-error` (sekcja 4); wyjątek od stronicowania dla serii czasowych (sekcja 5); nowe wartości wyliczeniowe i obowiązek klienta (sekcja 6). A-02 rozstrzygnięte, nowa A-03 |
