# Wymagania MVP: aplikacja dla trenera personalnego

- **Wersja:** 0.1 (szkic)
- **Data:** 2026-11-02
- **Status:** Do weryfikacji z klientem
- **Powiązane:** makiety ekranów (`docs/mockups/`), decyzje architektoniczne (`docs/adr/`)

## 1. Cel produktu

Trener personalny układa plany treningowe na kartce, w SMS-ach i PDF-ach. Nie ma jednego miejsca, w którym widzi, czy podopieczni trenują, jakie ciężary robią i jak zmienia się ich sylwetka. Podopieczny nie ma pod ręką planu ani historii swoich wyników.

MVP ma zastąpić kartkę i SMS-y jedną aplikacją mobilną: trener układa i przypisuje plany, podopieczny wykonuje je i zapisuje wyniki, a obie strony widzą postępy.

**Miary sukcesu MVP** (do potwierdzenia z klientem):

- trener prowadzi w aplikacji wszystkich aktywnych podopiecznych zamiast w SMS-ach i PDF-ach,
- podopieczni zapisują w aplikacji większość wykonanych treningów.

## 2. Użytkownicy i role

| Rola            | Kto to                                                                      | Jak dostaje konto                    |
| --------------- | --------------------------------------------------------------------------- | ------------------------------------ |
| **Trener**      | Właściciel - /ka siłowni prowadząca podopiecznych. W MVP jest jeden trener. | Konto zakładane przy wdrożeniu       |
| **Podopieczny** | Osoba trenująca według planu od trenera                                     | Zaproszenie mailem od trenera (W-06) |

## 3. Zakres

**W zakresie MVP:** 9 ekranów z makiet: logowanie, lista podopiecznych, profil podopiecznego, kreator planu, biblioteka ćwiczeń, zaproszenie podopiecznego, „Dziś”, trening w trakcie, postępy.

**Poza zakresem MVP:**

- wielu trenerów w jednej siłowni i rola właściciela,
- wersja przeglądarkowa (komputer),
- zdjęcia sylwetki,
- dieta, czat, płatności, kalendarz i rezerwacje sesji, powiadomienia push.

## 4. Słownik pojęć

| Pojęcie              | Znaczenie                                                                              |
| -------------------- | -------------------------------------------------------------------------------------- |
| **Plan**             | Program treningowy dla jednego podopiecznego na określoną liczbę tygodni               |
| **Dzień treningowy** | Część planu (np. „Dzień A: dół") przypisana do dnia tygodnia                           |
| **Ćwiczenie**        | Pozycja z biblioteki (np. „Przysiad ze sztangą”)                                       |
| **Seria planowana**  | Cel ustalony przez trenera: liczba powtórzeń lub czas, ciężar, RIR, przerwa            |
| **RIR**              | „Reps in reserve”, czyli ile powtórzeń zostaje „w zapasie” do upadku mięśniowego       |
| **Trening**          | Jedno wykonanie dnia treningowego przez podopiecznego, z faktycznie wykonanymi seriami |
| **Pomiar**           | Wpis z wagą ciała i obwodami z określonej daty                                         |
| **Rekord osobisty**  | Najlepszy wynik podopiecznego w danym ćwiczeniu (definicja: P-04)                      |
| **Frekwencja**       | Odsetek zaplanowanych treningów, które zostały wykonane w danym okresie                |

## 5. Wymagania funkcjonalne

Format: historyjka użytkownika + kryteria akceptacji. Kryteria opisują, **co** ma się dziać, a nie **jak** to zbudować. Każde kryterium musi dać się sprawdzić testem.

### 5.1 Konto i dostęp

#### W-01 Logowanie _(ekran 1)_

Jako trener lub podopieczny chcę zalogować się e-mailem i hasłem, żeby mieć dostęp do swoich danych.

- Poprawny e-mail i hasło → użytkownik jest zalogowany i trafia na swój ekran startowy (trener: lista podopiecznych, podopieczny: „Dziś”).
- Błędny e-mail lub błędne hasło → ten sam komunikat w obu przypadkach. Nie zdradzamy, czy konto istnieje.
- Konto zablokowane lub zarchiwizowane → logowanie odrzucone.
- Wielkość liter w e-mailu nie ma znaczenia.
- Użytkownik pozostaje zalogowany między uruchomieniami aplikacji, dopóki się nie wyloguje.

#### W-02 Reset hasła _(ekran 1, „Nie pamiętam hasła”)_

Jako użytkownik, który zapomniał hasła, chcę ustawić nowe, żeby odzyskać dostęp.

- Po podaniu e-maila system wysyła link do ustawienia nowego hasła, jeśli konto istnieje.
- Komunikat po wysłaniu formularza jest taki sam niezależnie od tego, czy konto istnieje.
- Link jest jednorazowy i ważny 1 godzinę.
- Po ustawieniu nowego hasła wszystkie dotychczasowe sesje użytkownika wygasają.

#### W-03 Wylogowanie _(zakładka „Konto”)_

Jako użytkownik chcę się wylogować, żeby nikt inny na moim telefonie nie miał dostępu do moich danych.

- Po wylogowaniu aplikacja wraca do ekranu logowania.
- Po wylogowaniu sesji z tego urządzenia nie da się już odnowić.

### 5.2 Podopieczni (trener)

#### W-04 Lista podopiecznych _(ekran 2)_

Jako trener chcę widzieć listę swoich podopiecznych ze statusem, żeby szybko zauważyć, kto wymaga uwagi.

- Lista pokazuje imię, informację o planie i status każdego podopiecznego.
- Statusy:
  - **W planie:** ma aktywny plan i trenuje regularnie,
  - **Brak treningu:** ma aktywny plan, ale nie zapisał treningu od co najmniej 7 dni,
  - **Koniec planu:** aktywny plan kończy się w ciągu 3 dni,
  - **Bez planu:** nie ma aktywnego planu.
- Filtry: **Aktywni** (domyślny), **Wymaga uwagi** (statusy „Brak treningu” i „Koniec planu”), **Zarchiwizowani**. Przy filtrze widać liczbę podopiecznych.
- Wyszukiwanie po imieniu, nazwisku lub e-mailu, bez rozróżniania wielkości liter.
- Lista jest stronicowana po 20 pozycji i domyślnie posortowana alfabetycznie po imieniu.

#### W-05 Profil podopiecznego _(ekran 3)_

Jako trener chcę zobaczyć w jednym miejscu plan, postępy i ostatnie treningi podopiecznego, żeby przygotować się do rozmowy z nim.

- Profil pokazuje: imię i nazwisko, cel, datę rozpoczęcia współpracy.
- Sekcja **aktywnego planu**: nazwa, bieżący tydzień z liczby tygodni, liczba wykonanych treningów z zaplanowanych do dziś.
- Kafelki: **waga** i **obwód pasa** (ostatni pomiar i zmiana od pierwszego), **frekwencja** z ostatnich 4 tygodni.
- **Ostatnie treningi:** 3 najnowsze, z datą i liczbą wykonanych ćwiczeń z zaplanowanych. Możliwość przejścia do pełnej historii.
- **Notatka trenera:** tekst do 2000 znaków, widoczny wyłącznie dla trenera.
- Brak pomiarów lub treningów → czytelny stan pusty zamiast zer.

#### W-06 Zaproszenie podopiecznego _(ekran 6)_

Jako trener chcę zaprosić podopiecznego mailem, żeby sam założył sobie konto.

- Formularz: imię i nazwisko (wymagane), e-mail (wymagany), cel (opcjonalny: redukcja, masa, siła, zdrowie/sprawność).
- Po wysłaniu podopieczny dostaje e-mail z linkiem do ustawienia hasła.
- Link jest jednorazowy i ważny 7 dni.
- Lista **oczekujących zaproszeń** pokazuje datę wysłania i czas do wygaśnięcia.
- **Wyślij ponownie** generuje nowy link, a poprzedni przestaje działać.
- E-mail należący już do podopiecznego → błąd z czytelnym komunikatem.
- Po ustawieniu hasła konto podopiecznego jest aktywne, a zaproszenie znika z listy oczekujących.

#### W-07 Archiwizacja podopiecznego

Jako trener chcę zarchiwizować podopiecznego, który zakończył współpracę, żeby nie zaśmiecał listy, ale żeby jego historia została.

- Zarchiwizowany podopieczny nie może się zalogować.
- Widać go tylko w filtrze **Zarchiwizowani**, z pełną historią.
- Trener może przywrócić podopiecznego z archiwum.

### 5.3 Biblioteka ćwiczeń (trener)

#### W-08 Przeglądanie biblioteki _(ekran 5)_

Jako trener chcę szybko znaleźć ćwiczenie, żeby dodać je do planu.

- Biblioteka zawiera ćwiczenia **wspólne** (dostarczone z aplikacją) i **własne** trenera.
- Każde ćwiczenie ma nazwę, partię mięśniową, sprzęt i opcjonalny link do filmu z techniką.
- Wyszukiwanie po nazwie, bez rozróżniania wielkości liter.
- Filtry po partii mięśniowej oraz **Tylko moje**.
- Lista jest stronicowana.

#### W-09 Własne ćwiczenie

Jako trener chcę dodać własne ćwiczenie, żeby używać w planach ruchów, których nie ma w bibliotece.

- Wymagane: nazwa i partia mięśniowa. Opcjonalne: sprzęt, link do filmu, opis.
- Nazwa własnego ćwiczenia jest unikalna w obrębie biblioteki trenera.
- Trener może edytować tylko własne ćwiczenia. Ćwiczeń wspólnych nie może zmieniać.
- Ćwiczenia użytego w jakimkolwiek planie nie można usunąć, tylko ukryć z biblioteki. Istniejące plany i historia pozostają bez zmian.

### 5.4 Plany treningowe (trener)

#### W-10 Tworzenie planu _(ekran 4)_

Jako trener chcę ułożyć plan z dni treningowych, ćwiczeń i serii, żeby podopieczny wiedział dokładnie, co ma robić.

- Plan ma: nazwę, przypisanego podopiecznego i liczbę tygodni (1–52).
- Plan składa się z 1–7 dni treningowych. Każdy dzień ma nazwę i przypisany dzień tygodnia.
- Dzień treningowy zawiera ćwiczenia z biblioteki w ustalonej kolejności.
- Każde ćwiczenie ma 1–10 serii planowanych. Seria to liczba powtórzeń **albo** czas w sekundach, opcjonalnie ciężar i RIR, oraz przerwa po serii.
- Plan nie zapisze się bez nazwy, podopiecznego i co najmniej jednego dnia z co najmniej jednym ćwiczeniem. Błędy są pokazane przy właściwych polach.
- Podopieczny ma najwyżej jeden aktywny plan. Aktywowanie nowego planu kończy poprzedni.

#### W-11 Edycja planu w trakcie

Jako trener chcę zmienić plan w trakcie jego trwania, żeby dopasować go do postępów podopiecznego.

- Zmiany dotyczą tylko przyszłych treningów.
- Wykonane treningi pokazują dokładnie to, co było zaplanowane i wykonane w dniu ich wykonania. Edycja planu nie zmienia historii.

### 5.5 Trening (podopieczny)

#### W-12 Ekran „Dziś” _(ekran 7)_

Jako podopieczny chcę po otwarciu aplikacji zobaczyć, co mam dziś zrobić, żeby nie szukać planu w SMS-ach.

- Ekran pokazuje dzisiejszą datę i pasek bieżącego tygodnia ze statusem każdego dnia: wykonany, zaplanowany, dzisiaj, wolny.
- Jeśli na dziś jest zaplanowany trening: nazwa dnia, lista ćwiczeń z seriami i powtórzeniami oraz przycisk **Zacznij trening**.
- Jeśli na dziś nic nie ma: informacja o najbliższym zaplanowanym treningu.
- Jeśli podopieczny nie ma aktywnego planu: informacja, że trener jeszcze go nie przypisał.
- Przypomnienie o pomiarze, jeśli ostatni jest starszy niż 7 dni.

#### W-13 Zapisywanie treningu _(ekran 8)_

Jako podopieczny chcę zapisywać wykonane serie w trakcie treningu, żeby trener widział moje wyniki.

- Dla każdego ćwiczenia widać: film z techniką (jeśli jest), cel z planu oraz pola na ciężar i liczbę powtórzeń każdej serii.
- W polu ciężaru jest podpowiedź z poprzedniego wykonania tego ćwiczenia.
- Serię można oznaczyć jako wykonaną, a ćwiczenie można pominąć.
- Podopieczny może dodać uwagę dla trenera do każdego ćwiczenia (do 500 znaków). Trener widzi ją w historii treningu.
- Zakończenie zapisuje cały trening naraz: albo wszystkie serie, albo nic.
- **Przerwij** pozwala zapisać trening jako częściowo wykonany albo go odrzucić.
- Krótkie zerwanie połączenia w trakcie treningu nie powoduje utraty wpisanych danych.

### 5.6 Postępy (podopieczny i trener)

#### W-14 Pomiary _(ekran 9, „Dodaj pomiar”)_

Jako podopieczny chcę zapisywać wagę i obwody, żeby widzieć zmiany sylwetki.

- Pomiar ma datę (domyślnie dzisiejszą), wagę w kg i opcjonalne obwody w cm: pas, biodra, klatka, ramię, udo.
- Waga musi mieścić się w zakresie 30–300 kg, a obwody w 10–250 cm. Wartości spoza zakresu są odrzucane z komunikatem.
- Jednego dnia może być najwyżej jeden pomiar. Ponowny zapis tego samego dnia nadpisuje poprzedni.

#### W-15 Wykresy i rekordy _(ekran 9)_

Jako podopieczny chcę zobaczyć wykres wagi i swoje rekordy, żeby widzieć, że idę do przodu.

- Zakładka **Waga**: wykres z ostatnich 8 tygodni.
- Zakładka **Obwody**: wykres wybranego obwodu z ostatnich 8 tygodni.
- Zakładka **Rekordy**: rekord osobisty dla każdego wykonanego ćwiczenia (definicja: P-04).
- Te same dane trener widzi w profilu podopiecznego (W-05).

## 6. Reguły dostępu

| Dane                          | Trener                                      | Podopieczny                         |
| ----------------------------- | ------------------------------------------- | ----------------------------------- |
| Lista i profile podopiecznych | swoich podopiecznych                        | —                                   |
| Notatka trenera               | odczyt i zapis                              | brak dostępu                        |
| Plany                         | tworzenie i edycja dla swoich podopiecznych | odczyt **własnego** aktywnego planu |
| Treningi                      | odczyt treningów swoich podopiecznych       | zapis i odczyt **własnych**         |
| Pomiary i rekordy             | odczyt u swoich podopiecznych               | zapis i odczyt **własnych**         |
| Biblioteka ćwiczeń            | wspólne: odczyt; własne: pełna edycja       | odczyt ćwiczeń z własnego planu     |

**Każda próba dostępu do cudzych danych** (np. podopieczny A pobiera trening podopiecznego B przez API) **kończy się odmową**, a odpowiedź nie zdradza, czy te dane istnieją.

## 7. Wymagania niefunkcjonalne

### Bezpieczeństwo

- **N-01** Hasła są przechowywane wyłącznie jako skróty algorytmem zalecanym przez OWASP Password Storage Cheat Sheet.
- **N-02** Minimalna długość hasła to 8 znaków, a maksymalna co najmniej 64. Bez wymogów składu znaków; hasła z list wycieków są odrzucane (OWASP).
- **N-03** Cała komunikacja z API odbywa się przez HTTPS.
- **N-04** Zablokowanie lub zarchiwizowanie konta odcina dostęp najpóźniej po 15 minutach.
- **N-05** Po 5 nieudanych próbach logowania na jedno konto w ciągu 15 minut kolejne próby są czasowo spowalniane albo blokowane.
- **N-06** Logi aplikacji nie zawierają haseł, tokenów ani danych pomiarowych.

### Prywatność (RODO)

- **N-07** Dane osobowe i pomiary ciała są przetwarzane i przechowywane na serwerach w UE.
- **N-08** Na żądanie podopiecznego trener może trwale usunąć jego konto i wszystkie powiązane dane.

### Wydajność i dostępność

- **N-09** 95% żądań do API obsługiwanych jest w czasie poniżej 500 ms przy 100 podopiecznych z rocznym okresem historii.
- **N-10** Docelowa dostępność usługi: 99,5% w skali miesiąca.
- **N-11** Kopia zapasowa bazy danych wykonywana jest codziennie i przechowywana przez co najmniej 7 dni. Odtworzenie kopii jest sprawdzone przed startem produkcyjnym.

### Utrzymanie i wdrażanie

- **N-12** Zmiany schematu bazy danych są wprowadzane wyłącznie przez wersjonowane migracje.
- **N-13** Każda zmiana w gałęzi głównej przechodzi automatyczny build i testy (CI).
- **N-14** Aplikacja jest uruchamiana jako kontener. Środowisko lokalne da się postawić jednym poleceniem.
- **N-15** Reguły dostępu z sekcji 6 są pokryte testami automatycznymi.
- **N-16** API jest opisane w specyfikacji OpenAPI, generowanej z kodu.

### Zgodność

- **N-17** Zmiany API są wstecznie zgodne z wersjami aplikacji mobilnej opublikowanymi w ostatnich 3 miesiącach.
- **N-18** Daty i czasy są przechowywane w UTC i wyświetlane w strefie czasowej Europe/Warsaw.
- **N-19** Interfejs i komunikaty błędów są w języku polskim. Jednostki: kilogramy i centymetry.

## 8. Otwarte pytania do klienta

| Nr   | Pytanie                                                                                                  | Dotyczy    |
| ---- | -------------------------------------------------------------------------------------------------------- | ---------- |
| P-01 | Po ilu dniach bez treningu podopieczny „wymaga uwagi”? Przyjęto 7.                                       | W-04       |
| P-02 | Ile dni przed końcem planu ma się pokazywać ostrzeżenie? Przyjęto 3.                                     | W-04       |
| P-03 | Jakie obwody mierzysz u podopiecznych? Przyjęto: pas, biodra, klatka, ramię, udo.                        | W-14       |
| P-04 | Co jest „rekordem” w ćwiczeniu: największy ciężar, czy najlepszy ciężar × powtórzenia?                   | W-15       |
| P-05 | Czy dni treningowe są przypięte do dni tygodnia, czy podopieczny robi „kolejny dzień” wtedy, kiedy może? | W-10, W-12 |
| P-06 | Czy używasz RIR, czy wolisz inną skalę intensywności (np. RPE) albo żadnej?                              | W-10       |
| P-07 | Skąd mają pochodzić filmy z techniką: Twoje nagrania czy linki do istniejących filmów?                   | W-08, W-13 |
| P-08 | Czy podopieczni trenują czasem bez zasięgu (np. siłownia w piwnicy)?                                     | W-13       |
