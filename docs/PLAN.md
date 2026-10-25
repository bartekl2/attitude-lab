# attitude-lab — plan architektury (v1)

> Status: **zaakceptowany kierunek, szczegóły do przeglądu w PR.**
> Kod powstaje dopiero po akceptacji `docs/conventions.md` (etap 0).

## Cel

Strumieniowanie danych IMU z iPhone'a (SwiftUI + CoreMotion, ~100 Hz) do Maca,
gdzie Python (numpy) liczy orientację kilkoma filtrami na kwaternionach.
Viewer w przeglądarce (three.js) pokazuje obok siebie modele sterowane każdym
filtrem oraz wykresy błędu względem referencji. Sesje można nagrać do CSV
i odtworzyć przez filtry.

## Decyzje podjęte

| Temat | Decyzja |
|---|---|
| Referencja | Kwaternion z `CMDeviceMotion` to **referencja, nie ground truth**. Prawdziwa prawda: dane syntetyczne + fizyczne testy (powrót do startu, obroty o 90° wzdłuż krawędzi, test statyczny). |
| Ramka referencji | `xArbitraryZVertical` |
| Zakres sensorów v1 | **Tylko 6DoF** (żyroskop + akcelerometr). Magnetometr jest wysyłany i nagrywany (dla przyszłych wersji), filtry go nie używają. |
| Filtry v1 | gyro-only (baseline), accel-only (baseline), komplementarny (Valenti 2015), Madgwick, MEKF. **Mahony przesunięty do etapu 8.** |
| EKF | **MEKF** (multiplicative / error-state), stan: δθ (3) + bias żyroskopu (3). Nie EKF na 4 składowych kwaternionu. |
| Transport | iPhone → Mac: **UDP**. Mac → przeglądarka: **WebSocket** (JSON). |
| Nagrywanie | **Tylko na Macu**, ze strumienia UDP. Straty wykrywane po `seq`, replay obsługuje dziury. Nagrywanie na telefonie: etap 8. |
| Viewer | ES modules + import map (three.js, uPlot), **bez bundlera**. |
| CI | **Tylko Python** (ruff, mypy, pytest). iOS CI: etap 8. |
| Kolejność | Najpierw matematyka i dane syntetyczne, potem iOS. |
| Sprzęt | **Otwarte** — patrz „Otwarte decyzje”. |

## 1. Struktura repo

```
attitude-lab/
├── README.md                # GIF z viewera, tabela wyników, quickstart w 3 komendach
├── Makefile                 # test, live, replay F=..., bench, report
├── .github/workflows/
│   └── python.yml           # ruff, mypy, pytest (iOS CI później)
├── protocol/
│   ├── PROTOCOL.md          # jedyne źródło prawdy o formacie pakietu
│   └── fixtures/            # golden pakiety .bin + oczekiwane wartości .json
├── ios/
│   ├── project.yml          # XcodeGen (opcjonalnie, żeby nie walczyć z .pbxproj)
│   ├── AttitudeLab/         # MotionCapture, PacketEncoder, UdpLink, UI
│   └── AttitudeLabTests/    # enkoder vs protocol/fixtures (uruchamiane lokalnie)
├── core/                    # pakiet Pythona (uv, src layout)
│   ├── pyproject.toml       # deps: numpy; extras: [server] websockets, [plot] matplotlib, [dev]
│   ├── src/attitude_lab/
│   │   ├── quat.py          # czysta matematyka kwaternionów
│   │   ├── frames.py        # konwersja iOS → konwencja core, wyrównanie ramek
│   │   ├── types.py         # ImuSample, ImuBatch
│   │   ├── filters/         # gyro_only, accel_only, complementary, madgwick, mekf
│   │   ├── sim/             # trajektorie + model sensorów (seedowany)
│   │   ├── metrics.py
│   │   ├── io/              # packet.py (numpy dtype), recording.py (CSV + sidecar JSON)
│   │   ├── pipeline.py      # Source → Filters → Sinks, wspólne dla live i replay
│   │   └── server/          # udp_source, ws_broadcast, clock_sync, cli
│   └── tests/
├── viewer/
│   ├── cube.html            # etap 3: jedna kostka sterowana kwaternionem Apple'a
│   ├── index.html           # etap 6: pełny viewer
│   └── src/                 # scene, charts, ws (ES modules)
├── recordings/
│   ├── README.md            # opis każdej sesji
│   └── samples/             # małe nagrania w gicie; duże w .gitignore / GitHub Releases
└── docs/
    ├── PLAN.md              # ten dokument
    ├── conventions.md       # ramki, znaki, jednostki — PRZED kodem filtrów
    ├── filters.md           # teoria, wyprowadzenia, odnośniki
    ├── evaluation.md        # metodologia + wyniki
    └── adr/                 # maks. 3 krótkie ADR-y (lista niżej)
```

**Zasada izolacji:** `quat`, `filters/`, `sim`, `metrics` zależą tylko od numpy
i nie wiedzą nic o sieci ani o iOS (pilnuje tego test importów). Konwencje
Apple'a żyją wyłącznie w `frames.py`.

### ADR-y (maks. 3)

1. `001-transport.md` — UDP z telefonu, WebSocket do przeglądarki.
2. `002-mekf.md` — MEKF zamiast EKF na składowych kwaternionu.
3. `003-raw-data.md` — na łączu i w CSV są surowe dane w jednostkach Apple'a;
   konwersja tylko w `core`, więc poprawka konwencji nie unieważnia nagrań.

## 2. Protokół

### iPhone → Mac: UDP

- **Brak blokowania kolejki:** przy utracie pakietu na Wi-Fi TCP wstrzymuje
  wszystko do retransmisji, a potem wypluwa paczkę starych próbek. Do podglądu
  na żywo lepiej zgubić próbkę niż ją opóźnić.
- **Pasmo znikome:** ~150 B × 100 Hz ≈ 15 kB/s.
- **Koszt UDP:** sami wykrywamy straty (`seq`) i synchronizujemy zegary
  (PING/PONG). Nagrania z Maca mogą mieć dziury; są oznaczone i replay je obsługuje.
- iOS: `Network.framework` (`NWConnection`). Python: `asyncio.DatagramProtocol`.
- Wykrywanie Maca: v1 — ręczne IP w apce (zapamiętane), Bonjour później.

### Mac → przeglądarka: WebSocket

JSON, decymacja do 30–60 Hz (wyświetlacz nie potrzebuje 100 Hz).
Komunikaty sterujące z przeglądarki: reset, wyrównanie, pauza, wybór źródła.

### Format pakietu (binarny, little-endian, wersjonowany)

| Pole | Typ | Uwagi |
|---|---|---|
| magic | 4 B `"ATLB"` | |
| version, type | u8, u8 | type: 1=SAMPLE, 2=HELLO, 3=PING, 4=PONG |
| flags | u16 | has_mag, mag_calibrated, stale_gyro/acc, ramka referencji |
| seq | u32 | wykrywanie strat i zmiany kolejności |
| count | u8 (+pad) | liczba próbek w datagramie; v1 = 1 (batching później bez zmiany wersji) |
| t_dm | f64 | `CMDeviceMotion.timestamp` [s od bootu] |
| q_ref | f32×4 | kwaternion Apple'a w kolejności Apple'a (x, y, z, w) |
| t_acc, acc | f64, f32×3 | **surowy** akcelerometr [g, znak Apple'a] |
| t_gyr, gyr | f64, f32×3 | **surowy** żyroskop [rad/s, z biasem] |
| t_mag, mag | f64, f32×3 | surowy magnetometr [µT] — nagrywany, w v1 nieużywany |
| rot_rate_dm | f32×3 | żyroskop z poprawką biasu od Apple'a (osobny eksperyment) |
| mag_cal, mag_acc | f32×3, i8 | skalibrowane pole od Apple'a + dokładność |
| gravity_dm | f32×3 | tylko do debugowania konwencji, **nigdy jako wejście filtra** |
| t_send | f64 | zegar telefonu przy wysyłce (latencja) |

- **HELLO** (co ~1 s, payload JSON): model urządzenia, wersja iOS, wersja apki,
  ramka referencji, zadana częstotliwość, wersja konwencji → trafia do sidecara nagrania.
- **PING/PONG:** synchronizacja zegarów w stylu NTP, offset z pomiarów o najmniejszym RTT.
- Te same golden fixtures z `protocol/fixtures/` testują enkoder w Swifcie i parser w Pythonie.

### Nagrania

CSV z surowymi wartościami (jak na łączu) + `*.meta.json` (HELLO, statystyki strat,
wersja protokołu i konwencji).

## 3. API filtrów (szkic)

```python
@dataclass(frozen=True)
class ImuSample:          # już w konwencji core: SI, specific force, ramka body z conventions.md
    t: float; gyro: Vec3; accel: Vec3; mag: Vec3 | None

class AttitudeFilter(Protocol):
    name: str
    params: Params                         # dataclass: beta / Q,R / max_gap ...
    def reset(self, q0: Quat | None = None) -> None: ...
    def update(self, s: ImuSample) -> Estimate: ...   # q + diagnostyka (bias, innowacja, accel_rejected)

def run(f: AttitudeFilter, batch: ImuBatch, q0=None) -> Trajectory   # offline, deterministyczne
```

- Wewnątrz każdego filtra jest **czysta funkcja** `step(state, sample, dt, params) -> state`;
  klasa to cienka nakładka. Łatwe testy i przeglądy parametrów.
- **dt z timestampów próbki**, nigdy z czasu przyjścia pakietu. Obsługa: pierwsza
  próbka, dt ≤ 0, przerwa > `max_gap` (pomiń całkowanie / ponowna inicjalizacja).
- Bez losowości w filtrach; losowość tylko w `sim`, z ziarnem.
- Live i replay używają **tego samego** `pipeline.py`; różni je tylko źródło
  (`UdpSource` / `CsvSource`).
- Kwaterniony w core: Hamilton, skalar pierwszy (w, x, y, z), q obraca body → świat.
  `scipy.spatial.transform.Rotation` wyłącznie jako wyrocznia w testach.

## 4. Testy i metryki

### Poziomy testów

1. **Jednostkowe:** matematyka kwaternionów (hypothesis + porównanie ze scipy),
   `frames.py` na znanych przypadkach, kodek pakietów (golden fixtures), zapis/odczyt CSV.
2. **Filtry na danych syntetycznych** (własności, nie dokładne liczby):
   - żyroskop bez szumu → dokładna integracja (< 1e-6 rad),
   - start ze złej orientacji → zbieżność pochylenia w czasie T,
   - MEKF usuwa bias żyroskopu,
   - błąd ograniczony przy przyspieszeniach liniowych,
   - wynik zgodny przy 50 / 100 / 200 Hz,
   - odporność na przerwy i jitter dt.
3. **Regresja na prawdziwych nagraniach:** replay `recordings/samples/` vs zapisane
   metryki (z tolerancją); dwa przebiegi identyczne; live i offline identyczne dla tego samego wejścia.
4. **Protokoły fizyczne** (opisane w `evaluation.md`): 10–30 min w bezruchu,
   4 × 90° wzdłuż krawędzi i powrót do startu, potrząsanie, chodzenie.

Dodatek: wariancja Allana z długiego nagrania statycznego → realne parametry
szumu do `sim` i do macierzy Q w MEKF.

### Metryki

W v1 (6DoF, ramka z dowolnym X) kurs jest nieobserwowalny dla filtrów, więc
**główna metryka to błąd pochylenia**. Błąd kursu służy do pomiaru dryfu.

| Metryka | Definicja |
|---|---|
| Błąd pochylenia [°] | kąt między kierunkiem grawitacji w ramce body: estymata vs referencja |
| Błąd kursu [°] | obrót wokół pionu; po wyrównaniu startowym |
| Błąd całkowity [°] | `2·atan2(‖v‖, |w|)` z `q_ref* ⊗ q_est`, po wyrównaniu kursu |
| Statystyki | RMSE, mediana, p95, max; osobno segmenty statyczne i dynamiczne |
| Dryf [°/min] | nachylenie błędu kursu w czasie (test statyczny) |
| Czas zbieżności | czas do < 2° po złym starcie |
| Opóźnienie estymacji [ms] | korelacja wzajemna estymata vs prawda (na syntetycznych) |
| Latencja | sensor → callback (telefon), telefon → Mac (sync zegarów), `update()` per filtr p50/p99, Mac → render |
| Jakość strumienia | % strat, jitter, rzeczywista częstotliwość z timestampów sensora |

Wyrównanie ramek (stały offset z pierwszych N sekund lub inicjalizacja z `q_ref`)
jest zawsze zapisywane w raporcie.

## 5. Etapy i kryteria „done”

Progi liczbowe to **hipotezy startowe**, kalibrowane po pierwszych prawdziwych danych.

| # | Etap | Done |
|---|---|---|
| 0 | Szkielet repo, `conventions.md`, CI Pythona, 3 ADR-y | **Akceptacja `conventions.md` przez autora** („rozumiem i akceptuję”); CI zielone na pustych testach |
| 1 | `quat`, `sim`, `metrics` | Testy własności przechodzą; integracja syntetycznego żyroskopu bez szumu < 1e-6 rad |
| 2 | Filtry offline, w kolejności: 2a gyro-only + accel-only → 2b komplementarny (Valenti) → 2c Madgwick → **2d MEKF (na końcu)** | Każdy filtr przechodzi zestaw scenariuszy syntetycznych z progami; `make report` generuje wykresy porównawcze |
| 3 | iOS + UDP + nagrywarka na Macu + **mini-viewer `cube.html`** | 10 min sesji, strata < 1%, zmierzona rzeczywista częstotliwość; golden test kodeka zielony w Swifcie i Pythonie; **kostka w przeglądarce obraca się zgodnie z telefonem** (kwaternion Apple'a przez minimalny przekaźnik UDP → WS) |
| 4 | Walidacja konwencji na prawdziwych danych | Statycznie, z inicjalizacją z `q_ref`: błąd pochylenia < 1°; znaki i kierunki potwierdzone testami „połóż płasko / obróć o 90°”; `conventions.md` zaktualizowany o wyniki |
| 5 | Pipeline live + replay + WS | Replay deterministyczny i identyczny z offline. **Hipoteza do weryfikacji:** p99 `update()` dla wszystkich filtrów łącznie < 2 ms; jeśli nie wychodzi, mierzymy i decydujemy |
| 6 | Pełny viewer | Stabilne 60 fps z 5 modelami + referencją; przełącznik pochylenie / kurs / całkowity; reset i wyrównanie; tryb replay |
| 7 | Benchmark + README | `make bench` odtwarza tabelę wyników; README z GIF-em i wnioskami |
| 8 | Później | Mahony, 9DoF z magnetometrem (kalibracja hard/soft-iron), nagrywanie na telefonie, iOS CI, Bonjour, strojenie parametrów, latencja glass-to-glass, filtr w Swifcie |

### Mini-viewer `cube.html` (etap 3)

Cel: jak najszybciej zobaczyć kwaternion Apple'a „na żywo” i ustalić mapowanie
ramek CoreMotion → three.js, zanim powstanie pełny pipeline.

- Minimalny przekaźnik w `server/` (UDP → WS), reużywa `io/packet.py`.
- Jedna kostka z oznaczonymi osiami (X/Y/Z telefonu), „ekran” na jednej ścianie.
- Przełączniki do eksperymentów: kolejność składowych, sprzężenie, zamiana osi Y/Z.
- Wynik (które ustawienie jest poprawne) trafia do `conventions.md` w etapie 4.

### Wytyczne do `conventions.md` (etap 0)

Dokument dla osoby, która nie robiła jeszcze mechaniki ani nawigacji inercyjnej:

- **Rysunki ASCII** ramek: telefon (body), świat (CoreMotion, Z w górę), three.js (Y w górę).
- **Konkretne przykłady**, np. „telefon leży ekranem do góry → akcelerometr Apple'a
  pokazuje z ≈ −1 g, a po konwersji do specific force +9.81 m/s² w osi z”;
  „obrót o 90° w lewo wokół pionu → kwaternion …”.
- **Słowniczek:** kwaternion, sprzężenie, ramka body/świat, specific force, bias,
  pochylenie (tilt), kurs (yaw/heading), dryf, podwójne pokrycie (q i −q) itd.
- Miejsca niepewne (kierunek `CMQuaternion`, znaki) są jawnie oznaczone jako
  **„do weryfikacji w etapie 3/4”**, a nie zgadywane.

## 6. Ryzyka i założenia, które mogą być błędne

1. **Konwencje to błąd nr 1.** Akcelerometr Apple'a ekranem do góry daje z ≈ −1 g
   (znak odwrotny do specific force z wyprowadzeń Madgwicka); kierunek `CMQuaternion`
   trzeba ustalić eksperymentalnie; kolejność składowych różni się między bibliotekami;
   three.js ma Y w górę, CoreMotion Z. Stąd kostka w etapie 3 i osobny etap 4.
2. **„Surowe” dane nie są do końca surowe.** Akcelerometr i żyroskop są po kalibracji
   fabrycznej. `gravity` i `userAcceleration` to wynik fuzji — nigdy jako wejście filtra.
   `rotationRate` z deviceMotion ma usunięty bias (ukrywa dryf) — tylko jako osobny eksperyment.
3. **Strumienie nie są zsynchronizowane.** Osobne timestampy dla akcelerometru,
   żyroskopu i magnetometru; przy 500°/s przesunięcie 5 ms ≈ 2,5°. Mitygacja: odczyt
   pull najnowszych próbek w każdym ticku deviceMotion, zapis wszystkich timestampów,
   ewentualna interpolacja.
4. **100 Hz to sufit `CMMotionManager` i nie jest gwarantowane.** dt zawsze z timestampów.
5. **Straty UDP w nagraniach.** Przy nagrywaniu tylko na Macu dziury są nieuniknione;
   mierzymy je i raportujemy. Jeśli przekroczą ~1% w benchmarkach, przyspieszamy
   nagrywanie na telefonie z etapu 8.
6. **Latencja zdominowana przez Wi-Fi** (power save iPhone'a → paczki opóźnień ~100 ms).
   Dla filtrów bardziej miarodajne jest opóźnienie estymacji niż latencja transportu.
7. **Viewer może wprowadzać w błąd.** Filtry 6DoF rozjadą się z referencją w kursie,
   nawet działając dobrze → domyślnie widok „wyrównany kurs”, jawny przełącznik.
8. **Ograniczenia iOS:** symulator nie ma sensorów; apka tylko na pierwszym planie
   (`isIdleTimerDisabled`); uprawnienie Local Network (`NSLocalNetworkUsageDescription`).
9. **Rozrost zakresu.** O wartości portfolio decydują metodologia i dobre README
   z wynikami, nie kolejny filtr.

## Otwarte decyzje

- **Sprzęt / konto Apple Developer:** do uzupełnienia (płatne konto czy darmowe Apple ID).
  Nie wpływa na architekturę; przy darmowym Apple ID apkę trzeba ponownie podpisywać
  co 7 dni.
