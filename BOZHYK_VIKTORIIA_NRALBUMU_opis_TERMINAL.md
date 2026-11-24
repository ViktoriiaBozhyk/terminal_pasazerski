# Temat 17 – Terminal pasażerski

**Autor:** Viktoriia Bozhyk, nr albumu 156302, grupa laboratoryjna 6
**Repozytorium GitHub:** [https://github.com/ViktoriiaBozhyk/terminal_pasazerski](https://github.com/ViktoriiaBozhyk/terminal_pasazerski)
**Język / środowisko:** C (C11), Linux, serwer `torus.uck.pk.edu.pl`

---

## 1. Opis zadania

Symulacja terminala pasażerskiego obsługującego pasażerów **odlatujących** i **przylatujących**.
Z terminala odlatują samoloty do **5 destynacji**: D1–D3 w strefie Schengen, D4–D5 poza strefą Schengen.
Do każdej destynacji przypisane jest jedno wyjście (gate) i flota **S_i** samolotów o pojemności **P_i** pasażerów
i dopuszczalnej masie bagażu podręcznego **M_i**.

### 1.1. Ścieżka pasażera odlatującego

`[Odprawa / ważenie bagażu] → [Kontrola bezpieczeństwa] → [Kontrola paszportowa*] → [Strefa Gate] → [Schody pasażerskie] → [Samolot]`
(* tylko loty D4–D5 poza strefę Schengen)

1. **Odprawa** – pasażer z losową masą bagażu podręcznego; jeśli masa > M_i, pasażer zostaje wycofany do terminala (koniec procesu, wpis w logu).
2. **Kontrola bezpieczeństwa** – 3 stanowiska działające równolegle:
   - na stanowisku jednocześnie maksymalnie **2 osoby**,
   - jeśli na stanowisku są 2 osoby, muszą być **tej samej płci**,
   - pasażer może przepuścić w kolejce maksymalnie **3 inne osoby** – po trzecim przepuszczeniu otrzymuje pierwszeństwo i musi zostać obsłużony jako następny,
   - pasażerowie z niebezpiecznymi przedmiotami (losowo, ok. 1%) nie przechodzą kontroli.
3. **Kontrola paszportowa** – dla lotów poza Schengen; ok. 0,5% pasażerów zostaje zawróconych.
4. **Strefa Gate** – pasażer czeka przy wyjściu swojej destynacji, aż dyspozytor ogłosi gotowość samolotu do boardingu.
5. **Boarding** – wejście przez schody pasażerskie o pojemności **K** (K < P_i). Pasażerowie **VIP** (ok. 1%) omijają kolejkę przy gate, ale nie mogą ominąć schodów.

### 1.2. Samolot (kapitan)

- Samolot odlatuje co **T** jednostek czasu albo wcześniej po sygnale od dyspozytora.
- Przed odlotem kapitan musi zagwarantować, że **na schodach nie ma żadnego pasażera** oraz że liczba pasażerów w samolocie **≤ P_i**.
- Po odlocie na jego miejsce przy danym gate natychmiast podstawiany jest kolejny samolot z floty S_i (jeśli jest dostępny).
- Samolot wraca po losowym czasie **T_i** z losową liczbą pasażerów przylatujących i po ich obsłużeniu może być ponownie użyty.

### 1.3. Ścieżka pasażera przylatującego

`[Wyjście z samolotu] → [Kontrola paszportowa*] → [Hala przylotów] → [Wyjście z terminala]`

- Do terminala prowadzą tylko **2 wejścia** – jednocześnie pasażerowie mogą wysiadać tylko z 2 samolotów, pozostałe czekają.
- Ok. 25% pasażerów (loty spoza Schengen) przechodzi kontrolę paszportową, przy tych samych stanowiskach co dla odlotów; ok. 0,5% nie jest wpuszczanych do hali przylotów – są rejestrowani w logu.

### 1.4. Sygnały

| Sygnał | Nadawca → odbiorca | Działanie |
|---|---|---|
| **SIGUSR1** (sygnał 1 dyspozytora) | Dyrektor/dyspozytor → Kapitan gate *i* | samolot odlatuje przed czasem T (po zejściu ostatniego pasażera ze schodów) |
| **SIGUSR2** (sygnał 1 dyrektora) | Dyrektor → Kapitan gate *i* | gate *i* zostaje zamknięty, pasażerowie czekający przy nim opuszczają terminal |
| **SIGINT** (sygnał 2 dyrektora) | Dyrektor → grupa procesów odprawy/generatora | wstrzymanie odprawy – nowi pasażerowie nie mogą wejść; samoloty kończą pracę po przewiezieniu wszystkich pasażerów |
| **SIGTERM** | Dyrektor → wszystkie procesy | kontrolowane zakończenie symulacji i usunięcie struktur IPC |

Dyrektor portu pełni również rolę dyspozytora (ogłasza gotowość boardingu i wysyła sygnał wcześniejszego odlotu).

### 1.5. Doprecyzowania względem treści tematu

- W treści zamiennie używane są symbole P i N dla pojemności samolotu – przyjmuję **P_i = maksymalna liczba pasażerów samolotu do destynacji i**, a warunek schodów to **K < P_i**.
- Kontrola paszportowa dotyczy pasażerów lotów D4–D5 (ok. 2/5 lotów; procent pasażerów wynika z losowania destynacji, zgodnie z założeniem ok. 25%).
- Czas symulacji jest skalowany: 1 minuta symulacji = konfigurowalna liczba milisekund.

---

## 2. Architektura

Każda rola jest osobnym programem uruchamianym przez `fork()` + `exec()`:

| Program | Rola |
|---|---|
| `dyrektor` | proces główny: wczytuje i waliduje parametry, tworzy struktury IPC, uruchamia pozostałe procesy, wysyła sygnały, sprząta IPC na końcu |
| `kapitan` | po jednym na gate – podstawianie samolotów, boarding, odlot, powrót z przylatującymi |
| `bezpieczenstwo` | 3 procesy – stanowiska kontroli bezpieczeństwa |
| `paszport` | stanowisko kontroli paszportowej (wspólne dla odlotów i przylotów) |
| `generator` | tworzy w losowych odstępach czasu procesy pasażerów |
| `pasazer_odl` | pasażer odlatujący |
| `pasazer_przyl` | pasażer przylatujący |

**Mechanizmy IPC i synchronizacji:**

- **pamięć dzielona** (`shmget/shmat`) – wspólny stan: kolejki do stanowisk, liczniki przepuszczeń, stan gate'ów, liczba osób na schodach i w samolotach,
- **semafory System V** (`semget/semop`) – ochrona pamięci dzielonej, pojemność schodów (K), wejścia do terminala (2), miejsca na stanowiskach kontroli,
- **kolejka komunikatów** (`msgget/msgsnd/msgrcv`) – komunikacja pasażer ↔ stanowisko kontroli / kapitan (np. ogłoszenie boardingu, wynik kontroli),
- **sygnały** (`sigaction`) – polecenia dyrektora/dyspozytora,
- **pliki** (`open/write/close`) – logi symulacji w `logs/symulacja.log` (zapis z blokadą, aby wpisy się nie przeplatały).

Brak aktywnego czekania (busy waiting) – procesy czekają na semaforach lub `msgrcv`.

**Walidacja danych:** wszystkie parametry (liczba pasażerów, P_i, M_i, K, T, S_i) są sprawdzane (zakresy, K < P_i, liczba procesów ≤ limitu użytkownika z `getrlimit(RLIMIT_NPROC)`); przy błędzie wyświetlany jest komunikat i prośba o ponowne podanie wartości.
**Obsługa błędów:** każda funkcja systemowa sprawdzana, błędy zgłaszane przez `perror()` / `errno` we własnej funkcji obsługi błędów.
**Prawa dostępu:** struktury IPC tworzone z prawami `0600`; po zakończeniu wszystkie są usuwane (`IPC_RMID`).

---

## 3. Testy

| # | Nazwa | Scenariusz | Oczekiwany wynik |
|---|---|---|---|
| **T1** | Pojemność i płeć na stanowisku kontroli | 300 pasażerów odlatujących o losowej płci, 3 stanowiska | W logach w żadnym momencie na stanowisku nie ma > 2 osób ani dwóch osób różnej płci. |
| **T2** | Limit przepuszczeń (brak zagłodzenia) | Duża przewaga jednej płci (np. 90% K / 10% M) | Żaden pasażer nie przepuścił więcej niż 3 osób; każdy pasażer zostaje obsłużony (brak zakleszczenia). |
| **T3** | Schody i pojemność samolotu | P_i = 10, K = 3, 100 pasażerów do jednej destynacji | Na schodach nigdy > K osób; w chwili odlotu na schodach 0 osób; w samolocie nigdy > P_i pasażerów. |
| **T4** | Wcześniejszy odlot (SIGUSR1) | Wysłanie sygnału do kapitana w trakcie boardingu | Samolot odlatuje z niepełną liczbą pasażerów dopiero po zejściu ze schodów ostatniego wchodzącego; podstawiany jest kolejny samolot; brak blokady pozostałych pasażerów. |
| **T5** | Zamknięcie gate (SIGUSR2) | Zamknięcie gate D2 przy oczekujących pasażerach | Pasażerowie przy D2 opuszczają terminal, pozostałe gate'y działają normalnie, brak procesów zawieszonych. |
| **T6** | Wstrzymanie odprawy (SIGINT) | Sygnał w trakcie symulacji | Nowi pasażerowie nie wchodzą na odprawę, samoloty przewożą już odprawionych i kończą pracę, symulacja kończy się poprawnie. |
| **T7** | Przyloty – 2 wejścia | 5 samolotów wraca jednocześnie | W danym momencie pasażerowie wysiadają maksymalnie z 2 samolotów, pozostałe czekają. |
| **T8** | Sprzątanie zasobów | Zakończenie normalne oraz przez SIGTERM / Ctrl+C | Po zakończeniu `ipcs` nie pokazuje struktur symulacji, `ps` nie pokazuje procesów zombie ani osieroconych. |

Wyniki testów weryfikowane na podstawie plików logów (skrypt sprawdzający oraz ręczna analiza).
