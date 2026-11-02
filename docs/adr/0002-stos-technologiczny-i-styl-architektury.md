# 2. Stos technologiczny i styl architektury backendu

- **Status:** Zaproponowana
- **Data:** 2026-11-02

## Kontekst

Budujemy backend (API REST) aplikacji do prowadzenia podopiecznych przez trenera personalnego. Dane są silnie powiązane relacjami: trener → podopieczni → plany → dni treningowe → ćwiczenia → serie, a do tego historia wykonanych treningów i pomiarów. W MVP jedynym klientem API jest aplikacja mobilna, a po MVP dojdzie klient przeglądarkowy. Musimy wybrać język i framework, bazę danych oraz styl architektury.

## Czynniki decyzyjne

- **Spójność danych:** zapis treningu (wiele serii naraz) musi się udać w całości albo wcale; dane mają więzy między sobą.
- **Czas dostarczenia MVP** przy małym zespole.
- **Kompetencje zespołu:** zespół najlepiej zna ekosystem Java/Spring.
- **Bezpieczeństwo:** uwierzytelnianie i autoryzacja dostępu do danych podopiecznych oparte na sprawdzonych komponentach.
- **Koszt utrzymania:** jak najmniej elementów do wdrożenia i monitorowania.

## Rozważane opcje

**Język i framework**
1. Java + Spring Boot
2. TypeScript + NestJS
3. Node.js + Express

**Baza danych**
1. PostgreSQL
2. MongoDB

**Styl architektury**
1. Monolit modułowy
2. Mikroserwisy

## Decyzja

- **Java 25 (LTS) i Spring Boot 4.x**, budowanie Mavenem.
- **PostgreSQL** jako jedyna baza danych.
- **Monolit modułowy:** jedna aplikacja i jedna baza, kod podzielony na moduły według obszarów domeny (np. `auth`, `clients`, `exercises`, `plans`, `workouts`, `progress`).

**Uzasadnienie:**
- Spring Boot dostarcza gotowe, utrzymywane komponenty do bezpieczeństwa (Spring Security), dostępu do danych (Spring Data JPA), walidacji i obsługi błędów. W Expressie te elementy trzeba składać z osobnych bibliotek. NestJS jest blisko pod tym względem, ale zespół ma w nim mniejsze doświadczenie, co wydłuża dostarczenie MVP.
- PostgreSQL zapewnia transakcje i więzy integralności, których wymaga model danych. MongoDB nie wnosi korzyści przy tak relacyjnych danych i stałym schemacie.
- Monolit modułowy minimalizuje koszt wdrożenia i utrzymania przy obecnej skali (jedna siłownia). Mikroserwisy wprowadziłyby komunikację sieciową, rozproszone transakcje i wiele wdrożeń bez realnej korzyści. Podział na moduły pozwala wydzielić usługę w przyszłości, jeśli pojawi się taka potrzeba.

## Konsekwencje

**Pozytywne**
- Jedno wdrożenie i jedna baza: proste transakcje, debugowanie i monitorowanie.
- Sprawdzone komponenty bezpieczeństwa zamiast własnych implementacji.

**Negatywne**
- Większe zużycie pamięci niż w przypadku aplikacji Node.js, czyli wyższy koszt serwera.
- Granice między modułami nie są wymuszane przez infrastrukturę. Trzeba ich pilnować w code review (ewentualnie testami architektury).
- Aktualizacje głównych wersji Javy i Spring Boot wymagają planowanej migracji.
