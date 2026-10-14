---
project: null  # TODO: nazwa projektu — do ustalenia przed /10x-prd
context_type: greenfield
created: 2026-10-09
updated: 2026-10-09
checkpoint:
 current_phase: 8
 phases_completed: [1, 2, 3, 4, 5, 6, 7]
 gray_areas_resolved:
  - topic: MVP scope
    decision: "Scope down: bez zapisów na wyścigi, jedna prosta reguła punktacji, akceptacja dołączenia później (v1: admin tworzy/aktywuje konta)"
  - topic: auth
    decision: "email + hasło; role admin i uczestnik; ranking tylko dla zalogowanych"
  - topic: primary persona
    decision: "Organizator/admin grupy ok. 10 znajomych (autor); uczestnicy są użytkownikami drugorzędnymi"
  - topic: pain category
    decision: "Brak widoczności progresu/regresu; niesprawiedliwy ranking przy nieobecnościach; dane uwięzione na papierze/w archiwum"
 frs_drafted: 8
 quality_check_status: warned
---

## Seed idea

Aplikacja webowa (backend Java, PostgreSQL) do prowadzenia sezonu wyścigów gokartowych wśród znajomych. Użytkownicy mają konta, dołączają do turnieju za akceptacją admina i zapisują się na konkretne wyścigi. Wyniki (trening, kwalifikacje, wyścig) wpisuje admin. Punkty sumują się za sezon, a klasyfikacje dodatkowe pozwalają wysoko wypaść osobom, które nie były na wszystkich wyścigach.

Sugestie, jeszcze niezatwierdzone: pierwszy etap to ręczne wpisywanie wyników i ranking sezonu. Import ze zdjęcia tabeli to ewentualny dodatek na koniec.

Do ustalenia: zasady punktacji, definicje klasyfikacji dodatkowych, liczba godzin tygodniowo, wybór frontendu.

## Vision & Problem Statement

Pain: po wyjazdach na gokarty nie widać realnego progresu/regresu kierowcy na przestrzeni wyścigów/Grand Prix (format: trening → kwalifikacje → wyścig). Wyniki przychodzą na papierowej kartce od organizatora toru i leżą w archiwum bez użycia. Punkty dziś nie są liczone.

Moment: po każdym wyjeździe, gdy grupa chce porównać wyniki z poprzednimi.

Cost today: brak ciągłości i porównań. Pomysł na Excel "poległ" (autor: bez konkretnego powodu, nie został dokończony). Ranking za samą pozycję mógłby krzywdzić dobrego kierowcę, który nie jeździ na każdym wyścigu (z ok. 10 osób nie każdy zawsze jest dostępny).

Insight (słowa autora): nie jest to must-have, ale ma być fajnym rozwinięciem wyjazdów. Dodatkowy cel autora: nauczyć się samodzielnego budowania aplikacji end-to-end.

> Socrates: Excel mógłby rozwiązać część problemu. Odpowiedź: Excel upadł, bo nie został dokończony, a nie z przyczyny technicznej; aplikacja jest świadomym wyborem także z powodu celu edukacyjnego. Zapisane uczciwie, bez dopisywania uzasadnienia.

## User & Persona

Primary: autor jako organizator (admin) grupy ok. 10 znajomych jeżdżących na gokarty mniej więcej raz w miesiącu. Wpisuje wyniki z papierowej kartki i prowadzi sezon. Uczestnicy (znajomi) oglądają ranking i swój progres.

## Access Control

Logowanie email + hasło. Dwie role: admin i uczestnik. Niezalogowany użytkownik nie widzi rankingu (widoczny tylko dla zalogowanych członków).

- Dołączanie do turnieju: użytkownik zgłasza chęć, admin akceptuje.
- Zapis na konkretny wyścig: samodzielny zapis członka turnieju (ewentualnie dodanie przez admina).
- Admin wpisuje wyniki (trening, kwalifikacje, wyścig) i zarządza sezonem.

## Success Criteria

### Primary
- Admin wpisuje wyniki jednego Grand Prix, a uczestnik widzi poprawny ranking sezonu z punktami.

### Secondary
- Widok progresu kierowcy między Grand Prix.

### Guardrails
- Poprawność liczenia punktów.
- Zwykły uczestnik nie może edytować wyników ani zatwierdzać kont.

## MVP first flow (v1, po scope-down)

1. Admin się loguje.
2. Tworzy sezon i Grand Prix.
3. Wpisuje wyniki z kartki (trening, kwalifikacje, wyścig) dla startujących uczestników.
4. System liczy punkty wg jednej prostej reguły.
5. Uczestnik loguje się i widzi ranking sezonu oraz swój progres.

Odłożone poza v1 (decyzja użytkownika): zapisy uczestników na wyścigi; klasyfikacje dodatkowe (po pierwszym działającym rankingu); akceptacja dołączenia do turnieju (v1: admin sam tworzy/aktywuje konta). Import wyników ze zdjęcia: opcjonalny dodatek na koniec (sugestia, niezatwierdzona).

## Timeline budget

mvp_weeks: 6 (szacunek podany przez użytkownika; dostępny czas zmienny, brak ustalonej liczby godzin tygodniowo). Cel terminu: 6 grudnia 2026 (propozycja z /10x-idea-check, niepotwierdzona jako hard_deadline).

## Timeline acknowledgment

Acknowledged on 2026-10-09: 6-week MVP estimate exceeds the 3-week default; user chose scope-down and gave 6 weeks, requires sustained effort with variable time and motivation.

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

## User Stories

### US-01: Admin wpisuje wynik Grand Prix

- **Given** Grand Prix jest utworzone, uczestnicy są dodani (dodawanie możliwe także w trakcie trwania Grand Prix)
- **When** admin dodaje wynik po zakończonym Grand Prix
- **Then** # TODO: oczekiwany rezultat (co widzi uczestnik w rankingu) — see Open Questions

## Business Logic

Aplikacja z wyników trzech sesji każdego Grand Prix (trening, kwalifikacje, wyścig) liczy kilka równoległych tabel sezonu, a nie jeden ranking.

Tabele (decyzja użytkownika):
1. **Tabela główna** — punkty za pozycję w wyścigu.
2. **Tabela bonusów** — najszybszy w treningu, kwalifikacjach i wyścigu oraz zysk pozycji między kwalifikacjami a metą wyścigu.
3. **Tabela dla osób z niepełną frekwencją** (sprawiedliwość) — ma pozwolić wysoko wypaść osobom, które nie były na wszystkich wyścigach. Mechanizm: do ustalenia.
4. **Tabela w stylu ELO** — punkty startowe plus punkty za zajęte miejsce, z uwzględnieniem siły przeciwników za i przed kierowcą albo sumy siły wszystkich w wyścigu. Jedna z tabel, nie jedyna.

Wejścia: wyniki sesji wpisane przez admina. Wyjście: tabele sezonu widoczne dla zalogowanych uczestników.

Uwaga: v1 był zawężony do "jednej prostej reguły" (Step 3); obecny zakres tabel jest szerszy — kolejność wdrożenia ustalona w Open Questions.

## Non-Functional Requirements

- Dane wyników nie są widoczne dla niezalogowanych.
- Ranking sezonu ładuje się w < 2 s dla ok. 10 osób lub więcej.

## Product framing

- product_type: web-app
- target_scale: { users: small }
- timeline_budget: { mvp_weeks: 6, hard_deadline: 2026-12-06, after_hours_only: true }
- Sonda skali (100×): użytkownik nie wie.

## Non-Goals

- Import wyników ze zdjęcia tabeli (OCR/AI) — opcjonalny dodatek na koniec, poza zakresem MVP.
- Samorejestracja z akceptacją admina i zapisy na wyścigi — v2 (v1: admin ręcznie tworzy konta).
- Tworzenie wielu sezonów — v2 (v1: jeden sezon na sztywno).
- Aplikacja mobilna / natywna — poza zakresem.

## Forward: tech-stack

Preferencje użytkownika (informacyjnie, nie do PRD): backend Java (doświadczenie głównie z Javą 8 w legacy), PostgreSQL, frontend obojętny. Aplikacja webowa.


## Quality cross-check

Status: warned (user accepted gaps). Gaps:

1. Nazwa projektu nie została podana — PRD frontmatter `project` będzie placeholderem.
2. US-01: brak części "Then" (co uczestnik widzi w rankingu po wpisaniu wyniku).
3. Zasady punktacji nie są ustalone (ile punktów za pozycje, definicje bonusów, w tym zysku pozycji) — Business Logic jest opisana bez wartości.
4. Mechanizm tabeli dla niepełnej frekwencji nie jest wybrany; tabela ELO ma tylko ogólny opis (siła przeciwników).
5. Zakres 4 tabel przekracza "jedną prostą regułę" z scope-down v1 i szacunek 6 tyg. — kolejność wdrażania tabel do ustalenia (propozycja: najpierw tabela główna).
6. Liczba godzin tygodniowo i wybór frontendu nieustalone.
7. Sonda skali 100×: użytkownik nie wie.


