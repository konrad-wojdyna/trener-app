# 3. Uwierzytelnianie klienta mobilnego: JWT w nagłówku Authorization

- **Status:** Zaproponowana
- **Data:** 2026-11-02
- **Powiązane:** ADR-0002 (stos technologiczny), ADR o refresh tokenach (do napisania), ADR o kliencie przeglądarkowym (po MVP)

## Kontekst

- MVP aplikacji dla trenera to wyłącznie aplikacja na telefon: trener i podopieczny logują się z telefonu.
- Po MVP planujemy też wersję na komputer, działającą w przeglądarce.
- Backend to jedno API REST (stos: ADR-0002). Chcemy, żeby to samo API obsługiwało w przyszłości oba typy klientów, bez osobnych endpointów biznesowych dla każdego z nich.
- Telefon ma bezpieczny magazyn na sekrety (Keychain na iOS, Keystore na Androidzie). Przeglądarka takiego magazynu nie ma: każdy skrypt na stronie, także wstrzyknięty przez atakującego (XSS), może odczytać `localStorage`.
- Stare wersje aplikacji mobilnej żyją na telefonach użytkowników miesiącami. Zmiana sposobu logowania musi więc być zgodna wstecz.
- Zespół zna JWT i Spring Security.

## Rozważane opcje

1. **JWT w nagłówku `Authorization: Bearer`** (krótko żyjący access token + refresh token).
2. **Sesja w ciasteczku `HttpOnly` dla wszystkich klientów,** także dla telefonu.
3. **Zewnętrzny dostawca tożsamości** (np. Keycloak, Auth0).

## Decyzja

Wybieramy **opcję 1**.

- Po zalogowaniu (`POST /api/auth/login`) API zwraca access token (JWT, ważny krótko) i refresh token.
- Aplikacja mobilna trzyma oba tokeny w bezpiecznym magazynie telefonu i wysyła access token w nagłówku `Authorization: Bearer <token>` przy każdym żądaniu.
- API weryfikuje podpis i ważność tokenu bez zapytania do bazy.
- Szczegóły refresh tokenów (czas życia, rotacja, przechowywanie, unieważnianie) opisze osobny ADR.
- Gdy dojdzie klient przeglądarkowy, przed tym samym API stanie **BFF** (Backend-For-Frontend): trzyma tokeny po stronie serwera, a przeglądarce daje tylko ciasteczko `HttpOnly`. Opisze to osobny ADR. API biznesowe się nie zmieni.

**Dlaczego nie opcja 2:** ciasteczka to mechanizm przeglądarki. W aplikacji mobilnej trzeba je obsługiwać ręcznie i nic nie zyskujemy. Do tego dochodzi ochrona przed CSRF, której przy nagłówku nie potrzeba.

**Dlaczego nie opcja 3:** na etapie MVP to dodatkowy koszt (opłaty albo utrzymanie kolejnego serwera) i zależność od dostawcy, przy prostych wymaganiach: logowanie e-mailem i hasłem dla jednej siłowni. Wracamy do tej opcji, gdy pojawią się wymagania typu „logowanie przez Google” albo 2FA.

## Konsekwencje

**Łatwiejsze:**
- Jedno API dla wszystkich przyszłych klientów, bez endpointów „pod telefon” i „pod komputer”.
- Bezstanowa weryfikacja tokenu: brak zapytania do bazy przy każdym żądaniu.
- Brak ryzyka CSRF dla klienta mobilnego: token nie jest dołączany automatycznie, tak jak ciasteczko w przeglądarce. Aplikacja dodaje go świadomie.
- Zespół zna ten mechanizm, więc wdrożenie jest szybsze.

**Trudniejsze:**
- Wylogowanie i zablokowanie konta nie działają natychmiast: wydany access token jest ważny do końca swojego czasu życia. Dlatego musi żyć krótko, a to wymusza refresh tokeny (osobny ADR).
- Trzeba obsłużyć wygasanie tokenu w aplikacji mobilnej (odświeżenie i ponowienie żądania).
- Wersja przeglądarkowa będzie wymagała dodatkowego komponentu (BFF), czyli więcej do wdrożenia i utrzymania.
