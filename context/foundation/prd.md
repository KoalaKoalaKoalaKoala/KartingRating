---
project: "KartingRating"
version: 1
status: draft
created: 2026-10-11
context_type: greenfield
product_type: web-app
target_scale:
 users: small
timeline_budget:
 mvp_weeks: 6
 hard_deadline: 2026-12-06
 after_hours_only: true
---

## Vision & Problem Statement

Pain: po wyjazdach na gokarty nie widać realnego progresu/regresu kierowcy na przestrzeni wyścigów/Grand Prix (format: trening → kwalifikacje → wyścig). Wyniki przychodzą na papierowej kartce od organizatora toru i leżą w archiwum bez użycia. Punkty dziś nie są liczone.

Moment: po każdym wyjeździe, gdy grupa chce porównać wyniki z poprzednimi.

Cost today: brak ciągłości i porównań. Pomysł na arkusz "poległ" (autor: bez konkretnego powodu, nie został dokończony). Ranking za samą pozycję mógłby krzywdzić dobrego kierowcę, który nie jeździ na każdym wyścigu (z ok. 10 osób nie każdy zawsze jest dostępny).

Insight (słowa autora): nie jest to must-have, ale ma być fajnym rozwinięciem wyjazdów. Dodatkowy cel autora: nauczyć się samodzielnego budowania aplikacji end-to-end.

> Socrates: arkusz mógłby rozwiązać część problemu. Odpowiedź: arkusz upadł, bo nie został dokończony, a nie z przyczyny technicznej; aplikacja jest świadomym wyborem także z powodu celu edukacyjnego.

## User & Persona

Primary: autor jako organizator (admin) grupy ok. 10 znajomych jeżdżących na gokarty mniej więcej raz w miesiącu. Wpisuje wyniki z papierowej kartki i prowadzi sezon. Uczestnicy (znajomi) oglądają ranking i swój progres.

## Success Criteria

### Primary
- Admin wpisuje wyniki jednego Grand Prix, a uczestnik widzi poprawny ranking sezonu z punktami.

### Secondary
- Widok progresu kierowcy między Grand Prix.

### Guardrails
- Poprawność liczenia punktów.
- Zwykły uczestnik nie może edytować wyników ani zatwierdzać kont.

## User Stories

### US-01: Admin wpisuje wynik Grand Prix

- **Given** Grand Prix jest utworzone, uczestnicy są dodani (dodawanie możliwe także w trakcie trwania Grand Prix)
- **When** admin dodaje wynik po zakończonym Grand Prix
- **Then** # TODO: oczekiwany rezultat (co widzi uczestnik w rankingu) — see Open Questions

## Functional Requirements

Numeracja po usunięciu FR „Admin tworzy sezon" (v1: jeden sezon na sztywno; tworzenie sezonów w v2).

- FR-001: Admin może się zalogować. Priority: must-have
 > Socrates: Counter-argument considered: "przy jednym adminie logowanie to koszt bez wartości." Resolution: kept; stands as written.
- FR-002: Admin może ręcznie tworzyć/aktywować konta uczestników. Priority: must-have
 > Socrates: Counter-argument considered: "samorejestracja z akceptacją i tak będzie potrzebna, więc praca podwójna." Resolution: user kept manual account creation in v1; self-registration with admin approval moved to v2.
- FR-003: Admin może utworzyć Grand Prix w sezonie (v1: jeden sezon na sztywno). Priority: must-have
 > Socrates: Counter-argument considered: "w v1 wystarczy jeden sezon na sztywno." Resolution: FR "Admin tworzy sezon" removed from v1; season creation deferred to v2.
- FR-004: Admin może wpisać wyniki treningu, kwalifikacji i wyścigu dla startujących; uczestników można dodawać także w trakcie trwania Grand Prix. Priority: must-have
 > Socrates: no counter-argument; stands as written.
- FR-005: System liczy punkty wg reguły punktacji (patrz Business Logic). Priority: must-have
 > Socrates: no counter-argument; stands as written.
- FR-006: Uczestnik może się zalogować i zobaczyć ranking sezonu. Priority: must-have
 > Socrates: no counter-argument; stands as written.
- FR-007: Admin może edytować/usuwać wyniki. Priority: must-have
 > Socrates: no counter-argument; stands as written.
- FR-008: Uczestnik może zobaczyć swój progres między Grand Prix. Priority: nice-to-have
 > Socrates: no counter-argument; stands as written.

## Non-Functional Requirements

- Dane wyników nie są widoczne dla niezalogowanych.
- Ranking sezonu ładuje się w < 2 s dla ok. 10 osób lub więcej.

## Business Logic

Aplikacja z wyników trzech sesji każdego Grand Prix (trening, kwalifikacje, wyścig) liczy kilka równoległych tabel sezonu, a nie jeden ranking.

Tabele (decyzja użytkownika):
1. **Tabela główna** — punkty za pozycję w wyścigu.
2. **Tabela bonusów** — najszybszy w treningu, kwalifikacjach i wyścigu oraz zysk pozycji między kwalifikacjami a metą wyścigu.
3. **Tabela dla osób z niepełną frekwencją** (sprawiedliwość) — ma pozwolić wysoko wypaść osobom, które nie były na wszystkich wyścigach. Mechanizm: do ustalenia.
4. **Tabela w stylu ELO** — punkty startowe plus punkty za zajęte miejsce, z uwzględnieniem siły przeciwników za i przed kierowcą albo sumy siły wszystkich w wyścigu. Jedna z tabel, nie jedyna.

Wejścia: wyniki sesji wpisane przez admina. Wyjście: tabele sezonu widoczne dla zalogowanych uczestników.

### Proponowane zasady punktacji (propozycja asystenta na prośbę użytkownika — do potwierdzenia)

1. **Tabela główna:** punkty za pozycję w wyścigu wg skali 25-18-15-12-10-8-6-4-2-1 (pozycje 1–10), dalsze pozycje 0. Suma za sezon.
2. **Tabela bonusów:** +1 najszybszy w treningu, +1 najszybszy w kwalifikacjach, +1 najszybszy w wyścigu; +2 za każdą pozycję zyskaną między kwalifikacjami a metą (utrata pozycji = 0, bez punktów ujemnych). Suma za sezon.
3. **Tabela frekwencji (sprawiedliwość):** średnia punktów z tabeli głównej na start. Kierowca pojawia się w tabeli po min. 3 startach (próg do potwierdzenia), więc nieobecność nie obniża wyniku.
4. **Tabela ELO:** start 1000 punktów. Po wyścigu każdy kierowca jest porównywany z każdym innym startującym: wynik rzeczywisty 1 (wyżej), 0 (niżej), oczekiwany z różnicy rankingów; zmiana = K × średnia z (rzeczywisty − oczekiwany), K = 32. Pokonanie silniejszych daje więcej, nieobecni nie tracą punktów.

## Access Control

Logowanie email + hasło. Dwie role: admin i uczestnik. Niezalogowany użytkownik nie widzi rankingu (widoczny tylko dla zalogowanych członków).

- Dołączanie do turnieju: użytkownik zgłasza chęć, admin akceptuje (v1: admin ręcznie tworzy/aktywuje konta; samorejestracja z akceptacją — v2).
- Zapis na konkretny wyścig: samodzielny zapis członka turnieju (ewentualnie dodanie przez admina) — v2.
- Admin wpisuje wyniki (trening, kwalifikacje, wyścig) i zarządza sezonem.

## Non-Goals

- Import wyników ze zdjęcia tabeli (OCR/AI) — opcjonalny dodatek na koniec, poza zakresem MVP.
- Samorejestracja z akceptacją admina i zapisy na wyścigi — v2 (v1: admin ręcznie tworzy konta).
- Tworzenie wielu sezonów — v2 (v1: jeden sezon na sztywno).
- Aplikacja mobilna / natywna — poza zakresem.

## Open Questions

1. ~~Nazwa projektu~~ — rozstrzygnięte: KartingRating.
2. **US-01 "Then": co uczestnik widzi w rankingu po wpisaniu wyniku?** — Owner: użytkownik. Bez tego kryterium akceptacji historia jest niepełna.
3. **Czy proponowane zasady punktacji (skala, bonusy, K=32, próg 3 startów) są akceptowane?** — Owner: użytkownik; propozycja w Business Logic, zweryfikować na archiwalnych wynikach.
4. ~~Mechanizm tabeli frekwencji i ELO~~ — zaproponowane (średnia na start; ELO parami), czeka na potwierdzenie w pkt 3.
5. **W jakiej kolejności wdrażać tabele?** Cztery tabele przekraczają zakres "jednej prostej reguły" z v1 i szacunek 6 tygodni; propozycja z shapingu: najpierw tabela główna. — Owner: użytkownik.
6. **Ile godzin tygodniowo jest dostępne i jaki frontend?** — Owner: użytkownik; frontend do rozstrzygnięcia w kolejnym kroku (dobór stacku).
7. **Jak zmieniłaby się reguła punktacji przy 100× większej skali?** — użytkownik nie wie; niski priorytet.
8. **Czy 2026-12-06 jest twardym terminem?** Pochodzi z propozycji, w notatkach niepotwierdzona jako hard_deadline. — Owner: użytkownik.
