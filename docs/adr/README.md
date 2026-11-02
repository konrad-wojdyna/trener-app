# Decyzje architektoniczne (ADR)

Rejestr istotnych decyzji architektonicznych projektu. Format i zasady: [ADR-0001](0001-zapisywanie-decyzji-architektonicznych.md).

## Rejestr

| Nr | Tytuł | Status |
|---|---|---|
| 0001 | [Zapisywanie decyzji architektonicznych](0001-zapisywanie-decyzji-architektonicznych.md) | Zaakceptowana |
| 0002 | [Stos technologiczny i styl architektury backendu](0002-stos-technologiczny-i-styl-architektury.md) | Zaproponowana |
| 0003 | [Uwierzytelnianie klienta mobilnego](0003-uwierzytelnianie-klienta-mobilnego.md) | Zaproponowana |

## Konwencje

- **Kiedy pisać ADR:** gdy decyzja wpływa na strukturę systemu, cechy niefunkcjonalne (bezpieczeństwo, wydajność, dostępność), zależności, interfejsy (API) albo sposób budowy i wdrażania. ADR powstaje w momencie podejmowania decyzji, a nie z wyprzedzeniem.
- **Nazwa pliku:** `NNNN-tytul-malymi-literami.md`, numery kolejne z zerami na początku.
- **Statusy:** Zaproponowana → Zaakceptowana albo Odrzucona; później ewentualnie Wycofana albo Zastąpiona przez ADR-NNNN.
- **Niezmienność:** zaakceptowanego ADR-a nie edytujemy (poza literówkami i linkami). Zmiana decyzji oznacza nowy ADR, a stary dostaje status „Zastąpiona przez ADR-NNNN”.

## Szablon

```markdown
# NNNN. Tytuł decyzji

- **Status:** Zaproponowana
- **Data:** RRRR-MM-DD
- **Powiązane:** (opcjonalnie) ADR-NNNN

## Kontekst
Jaki problem lub sytuacja wymaga decyzji?

## Czynniki decyzyjne
(opcjonalnie) Co jest dla nas ważne przy wyborze?

## Rozważane opcje
1. ...
2. ...

## Decyzja
Co wybieramy i dlaczego (także: dlaczego nie pozostałe opcje).

## Konsekwencje
Co dzięki temu staje się łatwiejsze, a co trudniejsze?
```
