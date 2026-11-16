# 4. Refresh tokeny: czas życia, rotacja i unieważnianie

- **Status:** Zaakceptowana
- **Data:** 2026-11-16
- **Powiązane:** ADR-0003 (uwierzytelnianie klienta mobilnego), [kontrakt API](../api.md) (sekcja 8.1), [wymagania](../wymagania.md) (W-01, W-02, W-03, W-07, N-04)

## Kontekst

- ADR-0003 wybrał krótko żyjący access token (JWT, weryfikowany po podpisie, bez zapytania do bazy) i refresh token, a szczegóły refresh tokenów odłożył do osobnej decyzji.
- Wydanego access tokenu nie da się unieważnić: jest ważny do końca swojego czasu życia.
- Wymagania określają zachowanie sesji:
  - W-01: użytkownik pozostaje zalogowany między uruchomieniami aplikacji, dopóki się nie wyloguje,
  - W-02: po ustawieniu nowego hasła wszystkie dotychczasowe sesje wygasają,
  - W-03: po wylogowaniu sesji z tego urządzenia nie da się już odnowić,
  - W-07 i N-04: zarchiwizowanie lub zablokowanie konta odcina dostęp najpóźniej po 15 minutach.
- Użytkownik może być zalogowany na kilku urządzeniach jednocześnie.
- Refresh token jest długo żyjącym sekretem przechowywanym na telefonie. Trzeba założyć, że może wyciec, i umieć wtedy ograniczyć szkody.
- Kontrakt API zawiera już endpointy `POST /api/auth/login`, `refresh` i `logout`.

## Czynniki decyzyjne

- **Odcinanie dostępu** w czasie wymaganym przez N-04.
- **Wygoda:** brak ponownego logowania przy regularnym używaniu aplikacji (W-01).
- **Ograniczenie skutków wycieku** refresh tokenu albo bazy danych.
- **Bezstanowa weryfikacja zwykłych żądań** (ADR-0003): baza danych tylko przy logowaniu i odnawianiu.
- **Zgodność z praktyką branżową:** OAuth 2.0 Security Best Current Practice ([RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)), OWASP Session Management Cheat Sheet.

## Rozważane opcje

**Postać refresh tokenu**
1. Losowy ciąg znaków, w bazie zapisany tylko jego skrót.
2. JWT podpisany przez serwer, weryfikowany po podpisie.

**Odnawianie**
1. Rotacja: każde odnowienie wydaje nowy refresh token, a ponowne użycie wykorzystanego tokenu unieważnia całą sesję.
2. Bez rotacji: ten sam refresh token przez cały czas życia sesji.

**Wygasanie sesji**
1. Tylko limit bezczynności.
2. Limit bezczynności i limit bezwzględny (wymuszone ponowne logowanie po określonym czasie od zalogowania).

## Decyzja

### Czas życia

- **Access token: 10 minut.** Spring Security przy weryfikacji JWT domyślnie dopuszcza 60 sekund różnicy zegarów (`JwtTimestampValidator`). 10 minut zostawia zapas względem 15 minut z N-04.
- **Refresh token: 30 dni od wydania.** Każde odnowienie wydaje nowy refresh token z nowym terminem, więc w praktyce jest to limit bezczynności: sesja wygasa po 30 dniach bez otwierania aplikacji.
- **Bez limitu bezwzględnego**, zgodnie z W-01.

### Postać i przechowywanie

- Refresh token to 32 bajty z `SecureRandom`, zakodowane jako tekst (Base64url). Nie zawiera żadnych danych.
- W bazie zapisujemy wyłącznie **skrót SHA-256** tokenu, nigdy sam token. Token ma 256 bitów losowości, więc szybki skrót bez soli jest bezpieczny. Pozwala też wyszukać wpis po unikalnym indeksie.
- Wpis w rejestrze refresh tokenów zawiera: skrót, konto, identyfikator sesji (rodziny), moment wydania, termin ważności i moment wykorzystania.

### Sesje (rodziny tokenów)

- Każde logowanie zaczyna **nową sesję** z własnym identyfikatorem (rodzinę tokenów). Tokeny wydawane przy kolejnych odnowieniach należą do tej samej sesji.
- Każde urządzenie ma osobną sesję.

### Odnowienie (`POST /api/auth/refresh`)

Serwer wykonuje kontrole w podanej kolejności. Pierwsza niespełniona kończy się odpowiedzią 401:

1. Wpis o skrócie tokenu istnieje.
2. Wpis nie jest wykorzystany. **Wykorzystany token oznacza, że ktoś ma jego kopię:** serwer usuwa wtedy całą sesję i zwraca 401.
3. Wpis nie wygasł.
4. Konto nie jest zarchiwizowane ani zablokowane.

Gdy wszystkie kontrole przejdą, serwer w jednej transakcji oznacza wpis jako wykorzystany, zapisuje nowy refresh token w tej samej sesji i zwraca nową parę tokenów.

### Unieważnianie

Unieważnienie polega na usunięciu wpisów z rejestru.

| Zdarzenie | Zakres |
|---|---|
| Wylogowanie (W-03) | sesja, do której należy przesłany refresh token |
| Wykrycie ponownego użycia | sesja, do której należy wykorzystany token |
| Ustawienie nowego hasła (W-02) | wszystkie sesje konta |
| Archiwizacja lub zablokowanie konta (W-07, N-04) | wszystkie sesje konta |
| Trwałe usunięcie konta (N-08) | wszystkie wpisy konta, razem z kontem |

Wydane access tokeny pozostają ważne do końca swojego czasu życia (najwyżej 10 minut).

### Sprzątanie

Zadanie uruchamiane raz na dobę usuwa wpisy po terminie ważności.

### Obowiązki aplikacji mobilnej

- Przechowuje oba tokeny w bezpiecznym magazynie systemu (Keychain, Keystore).
- Wysyła access token z każdym żądaniem, a refresh token wyłącznie do `POST /api/auth/refresh` i `POST /api/auth/logout`.
- Na 401 ze zwykłego żądania odnawia tokeny i ponawia żądanie.
- **Wykonuje najwyżej jedno odnowienie naraz.** Pozostałe żądania czekają na jego wynik. Dwa równoległe odnowienia tym samym tokenem zostałyby uznane za ponowne użycie i zakończyłyby sesję.
- Na 401 z odnowienia usuwa tokeny i pokazuje ekran logowania.

### Dlaczego nie pozostałe opcje

- **Refresh token jako JWT:** unieważnienie i tak wymaga rejestru w bazie, więc weryfikacja po podpisie nic nie oszczędza. Token weryfikowany wyłącznie po podpisie byłby nieodwołalny: przy rotacji zarchiwizowany użytkownik odnawiałby go bez końca.
- **Bez rotacji:** skradziony refresh token działałby równolegle z tokenem właściciela przez cały czas życia sesji i nic nie ujawniłoby kradzieży.
- **Limit bezwzględny:** wymuszałby okresowe logowanie także aktywnych użytkowników, co łamie W-01.

## Konsekwencje

**Łatwiejsze:**
- Zwykłe żądania nie odpytują bazy. Baza jest używana przy logowaniu i odnowieniu, mniej więcej raz na 10 minut aktywnego używania aplikacji.
- Wylogowanie, reset hasła i archiwizacja działają przez usunięcie wpisów, z dokładnie określonym zakresem (jedna sesja albo całe konto).
- Wyciek bazy nie ujawnia refresh tokenów (są tylko skróty), a wyciek pojedynczego tokenu jest wykrywany przy następnym użyciu przez właściciela.
- Wylogowanie z jednego urządzenia nie wpływa na pozostałe.

**Trudniejsze:**
- **Odcięcie dostępu nie jest natychmiastowe:** do 10 minut po archiwizacji, resecie hasła lub wylogowaniu wydany access token nadal działa. Natychmiastowe odcięcie wymagałoby zapytania do bazy przy każdym żądaniu.
- **Złodziej, który użyje skradzionego tokenu pierwszy, ma dostęp** do najbliższego odnowienia wykonanego przez właściciela.
- **Utrata odpowiedzi na odnowienie** (np. zerwane połączenie) zostawia aplikację ze starym, już wykorzystanym tokenem. Ponowienie kończy sesję i wymaga ponownego logowania. Możliwa zmiana w przyszłości: krótkie okno tolerancji dla poprzedniego tokenu.
- **Ponowne użycie tokenu starszego niż 30 dni nie jest wykrywane,** bo jego wpis został już usunięty. Taki token jest jednak odrzucany.
- **Odstępstwo od zalecenia OWASP:** brak limitu bezwzględnego. Ryzyko ograniczają bezpieczny magazyn telefonu, rotacja z wykrywaniem ponownego użycia i unieważnianie przy resecie hasła i archiwizacji.
- Aplikacja mobilna musi poprawnie obsłużyć jedno odnowienie naraz. Błąd w tym miejscu objawia się przypadkowymi wylogowaniami.
- Gdy dojdzie zmiana hasła przez zalogowanego użytkownika, powinna unieważniać sesje na pozostałych urządzeniach (OWASP).
- Wiązanie tokenów z urządzeniem (DPoP, [RFC 9449](https://www.rfc-editor.org/rfc/rfc9449)) świadomie pomijamy jako nadmiarowe przy obecnym poziomie ryzyka.
