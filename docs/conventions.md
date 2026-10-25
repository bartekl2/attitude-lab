# Konwencje: ramki, kwaterniony, jednostki

> **Status: DRAFT.** Czeka na „rozumiem i akceptuję” autora. Dopóki tego nie ma,
> nie powstaje żaden kod, który zależy od tych konwencji.
>
> Miejsca oznaczone **DO WERYFIKACJI (cube.html, etap 3)** to rzeczy, których
> dokumentacja Apple'a nie mówi wprost w tekście (czasem są tylko na obrazkach,
> czasem wcale). Nie zgadujemy ich, tylko sprawdzamy testami z sekcji 6.

## Jak czytać ten dokument

1. Najpierw **słowniczek** (sekcja 1). Bez niego reszta jest trudna.
2. Potem **nasze konwencje** (sekcja 2): tak liczy cały Python w `core/`.
3. Potem **Apple** (sekcja 3) i **przejścia** między światami (sekcja 4).
4. **Przykłady** (sekcja 5) to najlepszy sposób, żeby sprawdzić intuicję.
5. **Testy weryfikacyjne** (sekcja 6): co sprawdzimy telefonem w etapie 3.
6. **Czy rozumiem?** (sekcja 7): pytania kontrolne przed akceptacją.

---

## 1. Słowniczek

**Ramka (układ współrzędnych).** Trzy prostopadłe osie X, Y, Z zaczepione w jakimś
punkcie, względem których podajemy liczby.
*Przykład:* ta sama strzałka „w górę” ma inne liczby w różnych ramkach: w ramce
pokoju to (0, 0, 1), a w ramce telefonu leżącego na boku to np. (1, 0, 0).

**Ramka body (ramka telefonu, B).** Osie przyklejone do telefonu, obracają się razem z nim.
*Przykład:* oś +X body zawsze wychodzi z prawej krawędzi telefonu, niezależnie od tego,
jak go trzymasz.

**Ramka świata (W).** Osie nieruchome względem pokoju; u nas Z zawsze w górę.
*Przykład:* gdy obracasz telefon, jego osie body się zmieniają, a osie świata stoją w miejscu.

**Ramka referencji (CoreMotion reference frame).** Ramka świata, którą wybiera Apple
i względem której podaje orientację telefonu (`CMAttitude`).
*Przykład:* w `xArbitraryZVertical` oś Z jest pionowa, a X leży poziomo w kierunku
ustalonym przy starcie pomiaru (nie jest to północ).

**Orientacja (attitude).** Jak ramka body jest obrócona względem ramki świata.
*Przykład:* „telefon leży ekranem do góry, górną krawędzią w stronę okna” to opis orientacji.

**Kwaternion (jednostkowy).** Cztery liczby (w, x, y, z) o długości 1, które kodują
obrót o kąt θ wokół osi n: `q = (cos(θ/2), n·sin(θ/2))`.
*Przykład:* obrót o 90° wokół osi Z to `(cos 45°, 0, 0, sin 45°) = (0.7071, 0, 0, 0.7071)`;
brak obrotu to `(1, 0, 0, 0)`.

**Sprzężenie (q\*).** Kwaternion z odwróconym znakiem części wektorowej; oznacza obrót odwrotny.
*Przykład:* `(0.7071, 0, 0, 0.7071)* = (0.7071, 0, 0, −0.7071)`, czyli 90° wokół Z w drugą stronę.

**Podwójne pokrycie (q i −q).** `q` i `−q` opisują dokładnie ten sam obrót.
*Przykład:* `(1, 0, 0, 0)` i `(−1, 0, 0, 0)` to oba „brak obrotu”; dlatego przy liczeniu
błędu używamy `|w|`, a nie `w`.

**Hamilton vs JPL.** Dwie umowy co do mnożenia kwaternionów; w Hamiltonie `i·j = k`,
w JPL `i·j = −k`.
*Przykład:* ten sam ciąg czterech liczb oznacza w nich obroty w przeciwne strony,
więc mieszanie konwencji daje błąd „wszystko się kręci odwrotnie”.

**Prędkość kątowa (ω) i reguła prawej dłoni.** Jak szybko i wokół której osi coś się
obraca, w rad/s; kciuk prawej dłoni wzdłuż osi, palce pokazują dodatni kierunek obrotu.
*Przykład:* telefon leży ekranem do góry i kręcisz nim przeciwnie do wskazówek zegara
(patrząc z góry) → ω_z > 0.

**Specific force (siła właściwa, f).** To, co naprawdę mierzy akcelerometr: przyspieszenie
minus grawitacja, `f = a − g`.
*Przykład:* telefon leży na stole (a = 0), grawitacja ciągnie w dół, więc
`f = 0 − (0, 0, −9.81) = (0, 0, +9.81)` — akcelerometr „czuje” stół pchający go w górę.
W swobodnym spadku `f = 0`.

**g0.** Standardowe przyspieszenie ziemskie, u nas `g0 = 9.80665 m/s²`.
*Przykład:* Apple podaje akcelerometr w jednostkach g, więc 1.0 g → 9.80665 m/s².

**Bias żyroskopu.** Stały (albo wolno pływający) fałszywy odczyt żyroskopu w bezruchu.
*Przykład:* telefon leży, a żyroskop pokazuje 0.005 rad/s wokół Z; po minucie całkowania
to już ok. 17° fałszywego obrotu.

**Dryf.** Powolne narastanie błędu orientacji w czasie, głównie przez bias żyroskopu.
*Przykład:* filtr gyro-only po 10 minutach w bezruchu „myśli”, że telefon obrócił się o kilkadziesiąt stopni.

**Tilt (pochylenie).** Jak bardzo telefon jest odchylony od poziomu, czyli wszystko poza
obrotem wokół pionu.
*Przykład:* podniesienie górnej krawędzi o 30° to tilt 30°. Akcelerometr widzi tilt, bo widzi grawitację.

**Yaw (kurs, heading).** Obrót wokół osi pionowej świata.
*Przykład:* kręcenie telefonem leżącym płasko na stole zmienia tylko yaw. Akcelerometr
tego nie widzi, bo grawitacja dalej celuje prosto w dół.

**6DoF / 9DoF.** Liczba osi sensorów: 6 = żyroskop + akcelerometr, 9 = jeszcze magnetometr.
*Przykład:* w v1 używamy 6DoF, więc yaw da się tylko całkować z żyroskopu (i dryfuje).

**Kąty Eulera (roll, pitch, yaw).** Trzy kolejne obroty wokół osi. Wygodne do pokazania
człowiekowi, ale mają osobliwości (gimbal lock).
*Przykład:* w core ich **nie używamy** do obliczeń; najwyżej do wyświetlania.

---

## 2. Konwencje core (Python, `core/`)

### 2.1 Ramka body (B)

Taka sama jak ramka urządzenia Apple'a (patrz sekcja 3.1), żeby nie przestawiać osi:

```
                 +Y  (górna krawędź)
                  ^
                  |
            .-----------.
            |  _______  |
            | |       | |
            | |       | |
            | | ekran | | ----> +X  (prawa krawędź)
            | |       | |
            | |_______| |
            |     o     |
            '-----------'

   +Z wychodzi z ekranu prosto w Twoją stronę  (⊙ = strzałka „na Ciebie”)
```

### 2.2 Ramka świata (W)

- **+Z w górę** (przeciwnie do grawitacji).
- **+X poziomo**, w kierunku ustalonym przy starcie (jak w `xArbitraryZVertical`), potem stały.
- **+Y = Z × X** (ramka prawoskrętna): patrząc z góry, Y jest 90° w lewo od X.

```
   Świat widziany z góry (oś +Z wychodzi z kartki w Twoją stronę: ⊙)

                 +Y_W
                  ^
                  |
                  |
         ---------⊙---------> +X_W     (kierunek X wybrany przy starcie)
                  |
                  |

   Dodatni obrót wokół +Z (reguła prawej dłoni), patrząc z góry:

                .--<--.
               /       \
              |    ⊙    |   = przeciwnie do ruchu wskazówek zegara
               \       /
                '-->--'
```

Grawitacja w świecie: `g_W = (0, 0, −g0)`.

### 2.3 Kwaterniony

| Umowa | Wybór |
|---|---|
| Mnożenie | **Hamilton**: `i² = j² = k² = ijk = −1`, `i·j = k` |
| Kolejność w tablicy | **(w, x, y, z)** — skalar pierwszy, `q[0] = w` |
| Znaczenie | **q obraca body → świat**: `v_W = q ⊗ v_B ⊗ q*`, czyli `v_W = R(q) · v_B` |
| Norma | zawsze 1 (normalizujemy po każdym kroku filtra) |
| Znak | `q` i `−q` równoważne; do wyświetlania i porównań wybieramy `w ≥ 0` |

Co znaczy „q obraca body → świat” w praktyce: bierzesz współrzędne strzałki
w ramce telefonu, mnożysz przez `R(q)` i dostajesz jej współrzędne w ramce pokoju.

*Przykład:* telefon leży ekranem do góry, obrócony o 90° w lewo (patrząc z góry):
`q = (0.7071, 0, 0, 0.7071)`. Prawa krawędź telefonu w body to (1, 0, 0).
`R(q)·(1, 0, 0) = (0, 1, 0)`, czyli prawa krawędź wskazuje teraz +Y świata.

Wzory referencyjne (do testów, nie trzeba ich pamiętać):

```
Iloczyn Hamiltona:
  (w1, v1) ⊗ (w2, v2) = (w1·w2 − v1·v2,  w1·v2 + w2·v1 + v1 × v2)

Macierz obrotu body → świat dla q = (w, x, y, z):
  R(q) = | 1−2(y²+z²)   2(xy−wz)     2(xz+wy)  |
         | 2(xy+wz)     1−2(x²+z²)   2(yz−wx)  |
         | 2(xz−wy)     2(yz+wx)     1−2(x²+y²)|

Całkowanie żyroskopu (ω w ramce body, dt w sekundach):
  q_{k+1} = q_k ⊗ Δq,   Δq = (cos(|ω|dt/2),  ω/|ω| · sin(|ω|dt/2))
```

### 2.4 Jednostki (SI) i sensory w core

| Wielkość | Jednostka | Znaczenie |
|---|---|---|
| czas `t` | s (float64) | |
| kąt | rad | stopnie **tylko** w UI i raportach |
| `gyro` | rad/s | ω w ramce body, reguła prawej dłoni |
| `accel` | m/s² | **specific force** w ramce body |
| `mag` | µT | v1: nieużywany |

W bezruchu: `accel = R(q)ᵀ · (0, 0, +g0)`, czyli „wektor do góry widziany przez telefon”.

### 2.5 Dlaczego akurat te konwencje

- **Hamilton**: standard w three.js, scipy, Eigen i ROS. Jedna konwencja od początku
  do końca oznacza mniej sprzężeń w losowych miejscach.
- **(w, x, y, z)**: zgadza się z zapisem matematycznym `q = w + xi + yj + zk`.
  scipy i three.js trzymają (x, y, z, w); przestawiamy kolejność **tylko na granicach**
  (scipy wyłącznie w testach, three.js w viewerze).
- **body → świat**: tak działa `object.quaternion` w three.js (obiekt → rodzic), a całkowanie
  żyroskopu to proste mnożenie z prawej strony (`q ⊗ Δq`), bo żyroskop mierzy ω w body.
- **SI**: nie mieszamy g z m/s² ani stopni z radianami w obliczeniach.
- **Specific force**: to fizycznie mierzy akcelerometr i tak zapisane są wyprowadzenia
  Madgwicka, Valentiego i MEKF.

Artykuły używają różnych notacji (np. raport Madgwicka opisuje q jako orientację ramki
Ziemi względem czujnika). Dla każdego filtra zapiszemy w `docs/filters.md`, jak jego notacja
mapuje się na naszą, i sprawdzimy to testem na danych syntetycznych.

---

## 3. Konwencje Apple'a (CoreMotion)

Legenda: **[DOK]** = napisane w tekście dokumentacji Apple'a;
**[OBRAZEK]** = tylko na ilustracji w dokumentacji;
**DO WERYFIKACJI** = nie wynika z tekstu, sprawdzamy w etapie 3.

### 3.1 Osie urządzenia

X w prawo, Y do górnej krawędzi, Z z ekranu (jak na rysunku w 2.1).
**[OBRAZEK]**, wysoka pewność, ale **DO WERYFIKACJI (cube.html, etap 3)** — test V2.

### 3.2 Akcelerometr (`CMAccelerometerData.acceleration`)

- Jednostki: **g** **[DOK]**. Apple podaje 1 g = 9.8 m/s² w jednym miejscu i 9.81 m/s²
  w innym; my używamy `g0 = 9.80665`. Różnica (< 0.1%) zmienia tylko długość wektora,
  nie jego kierunek, więc nie wpływa na tilt.
- Znak: tekst mówi tylko, że wartości mogą być dodatnie lub ujemne.
  **Hipoteza:** Apple raportuje `−f / g0` (telefon ekranem do góry → z ≈ −1).
  **DO WERYFIKACJI (cube.html, etap 3)** — test V1.
- Fizyka pomaga: akcelerometr nie odróżnia grawitacji od przyspieszenia, więc znak jest
  jeden dla obu. Jeden test statyczny ustala go na zawsze: `acc_apple = s · f / g0`, gdzie `s = ±1`.

### 3.3 Żyroskop

- Surowy (`CMGyroData.rotationRate`): **rad/s** **[DOK]**; może mieć bias, np. od temperatury **[DOK]**.
- `CMDeviceMotion.rotationRate`: Apple usuwa bias **[DOK]**. W v1 tylko do eksperymentu porównawczego.
- Kierunek obrotu (czy reguła prawej dłoni): **[OBRAZEK]**, **DO WERYFIKACJI (cube.html, etap 3)** — test V3.

### 3.4 Grawitacja z fuzji (`CMDeviceMotion.gravity`)

- Wektor grawitacji w ramce urządzenia, w g **[DOK]**.
- Całkowite przyspieszenie = `gravity + userAcceleration` **[DOK]**.
- Znak (czy ekranem do góry daje (0, 0, −1)): **DO WERYFIKACJI** — wynika z V1.
- Nigdy jako wejście filtra (to już wynik fuzji Apple'a). Tylko do debugowania i testu V5.

### 3.5 Ramka referencji `xArbitraryZVertical`

- Z pionowa, X w dowolnym kierunku w płaszczyźnie poziomej **[DOK]**.
- Przy starcie Core Motion ustawia ramkę na podstawie początkowej orientacji urządzenia **[DOK]**.
- Czy Z wskazuje w **górę**, czy w dół: tekst mówi tylko „pionowa”. **DO WERYFIKACJI** — test V2.
- Czy na starcie yaw = 0 (q ≈ czysty tilt): **DO WERYFIKACJI**. Nie jest krytyczne, bo i tak wyrównujemy yaw przed porównaniem.

### 3.6 Kwaternion (`CMAttitude.quaternion`, typ `CMQuaternion`)

- Pola: `x, y, z, w` (Double) **[DOK]**. Na łączu wysyłamy je w tej kolejności (plan, sekcja 2).
- Znaczenie: `q = x·i + y·j + z·k + w`, obrót o θ wokół osi `{x, y, z}`,
  `w = cos(θ/2)`, `(x, y, z) = oś · sin(θ/2)` **[DOK]**.
- Opisuje orientację urządzenia względem ramki referencji **[DOK]**.
- **Nie wiadomo z tekstu:**
  1. czy to Hamilton, czy JPL,
  2. czy obraca urządzenie → referencję, czy odwrotnie.

  Dla nas oba pytania sprowadzają się do jednego: `q_core = (w, x, y, z)` **albo**
  `q_core = (w, −x, −y, −z)` (sprzężenie). **DO WERYFIKACJI (cube.html, etap 3)** — test V4.

### 3.7 Czas

- `timestamp` każdej próbki (akcelerometr, żyroskop, deviceMotion): sekundy od uruchomienia
  urządzenia **[DOK]**. Używamy tylko różnic (dt).
- `t_send` z apki (`ProcessInfo.systemUptime`) na tej samej osi czasu co `timestamp`:
  **DO WERYFIKACJI** — test V6.

### 3.8 Magnetometr

µT. W v1 tylko nagrywany, filtry go nie używają.

---

## 4. Tabele przejść

### 4.1 iOS → core (`frames.py`)

`s` to znak akcelerometru z testu V1 (hipoteza: `s = −1`).

| Wielkość | Na łączu (Apple) | W core | Przejście w `frames.py` | Status |
|---|---|---|---|---|
| osie body | X prawo, Y góra, Z z ekranu | to samo | bez zmian | DO WERYFIKACJI (V2) |
| ramka świata | Z pionowa, X dowolna pozioma | Z w górę, X dowolna pozioma | bez zmian, jeśli Z Apple'a = w górę | DO WERYFIKACJI (V2) |
| `acc` → `accel` | g, `s·f/g0` | m/s², specific force | `accel = s · g0 · acc` (hipoteza: `−g0 · acc`) | DO WERYFIKACJI (V1) |
| `gyr` → `gyro` | rad/s, z biasem | rad/s, prawa dłoń | bez zmian | jednostki [DOK]; znak DO WERYFIKACJI (V3) |
| `rot_rate_dm` | rad/s, bez biasu | jak `gyro` | bez zmian | jak wyżej |
| `gravity_dm` | g, ramka urządzenia | tylko debug | `· g0` | znak DO WERYFIKACJI (V1) |
| `q_ref` | (x, y, z, w) | (w, x, y, z), body → świat | przestawienie kolejności; **może** też sprzężenie | kolejność [DOK]; kierunek DO WERYFIKACJI (V4) |
| `t_*` | s od bootu | s | bez zmian | [DOK] |
| `mag` | µT | µT | bez zmian | v1 nieużywany |

### 4.2 core → three.js (viewer)

three.js: ramka prawoskrętna, **+Y w górę**, domyślna kamera patrzy wzdłuż −Z,
kwaternion zapisany jako `new THREE.Quaternion(x, y, z, w)`, `object.quaternion`
obraca obiekt → rodzic (czyli tak jak nasze body → świat).

```
   core (Z w górę)                    three.js (Y w górę)

        +Z                                 +Y
         ^                                  ^
         |   +Y (w głąb kartki)             |
         |  /                               |
         | /                                |
         +------> +X                        +------> +X
                                           /
                                          /
                                        +Z (w stronę kamery)
```

| | core | three.js |
|---|---|---|
| oś w górę | +Z | +Y |
| punkt / wektor | (x, y, z) | (x, z, −y) |
| zamiana ramek świata | — | obrót o −90° wokół X |
| kwaternion telefonu | (w, x, y, z) | `phone.quaternion.set(x, y, z, w)` **wewnątrz grupy obróconej o −90° wokół X** |

Zalecany sposób (prosty do zrozumienia):

1. W scenie tworzymy grupę `world` z `rotation.x = −π/2`. Ta grupa „jest” naszą ramką świata core.
2. Model telefonu budujemy w osiach body: szerokość wzdłuż X, wysokość wzdłuż Y,
   grubość wzdłuż Z, ekran na ścianie +Z.
3. Telefon wkładamy do grupy `world` i podajemy mu `q_core`, tylko z przestawioną kolejnością.

Równoważnie bez grupy: `q_three = q_M ⊗ q_core`, gdzie `q_M = (0.7071, −0.7071, 0, 0)`
(w, x, y, z), czyli −90° wokół X.

*Sprawdzenie:* `q_core = (1, 0, 0, 0)` (telefon płasko, ekranem do góry) → w three.js
ekran patrzy w +Y (do góry sceny), a górna krawędź w −Z (od kamery). Zgadza się z rzeczywistością.

---

## 5. Przykłady

Wartości „Apple” zakładają hipotezy (`s = −1`, prawa dłoń, Z w górę) do potwierdzenia
w sekcji 6. Wartości „core” wynikają z naszych konwencji i nie zależą od Apple'a.
Wszystkie kwaterniony podane dla yaw = 0; przy innym yaw dochodzi obrót wokół Z świata.

### 5.1 Leży ekranem do góry, nieruchomo

```
  Widok z boku

         +Z_W (w górę)                +Z_B (z ekranu)
            ^                            ^
            |                            |
     .------|----------------------------|------.
     |=============== ekran ====================|
     '------------------------------------------'
  ===================== stół =========================
                      |
                      v  grawitacja
```

| | Apple (surowe) | core |
|---|---|---|
| accel | (0, 0, −1) g | (0, 0, +9.81) m/s² |
| gyro | ≈ (0, 0, 0) + bias | ≈ (0, 0, 0) + bias |
| gravity_dm | (0, 0, −1) g | — |
| q | do ustalenia (V4) | (1, 0, 0, 0) |

### 5.2 Leży ekranem do góry, obrócony o 90° w lewo (patrząc z góry)

| | core |
|---|---|
| accel | (0, 0, +9.81) — **bez zmian względem 5.1** |
| q | (0.7071, 0, 0, 0.7071) |
| prawa krawędź (body +X) w świecie | (0, 1, 0) |

Wniosek: akcelerometr nie widzi obrotu wokół pionu. Stąd w 6DoF yaw tylko z żyroskopu.

### 5.3 Leży ekranem w dół, nieruchomo

| | Apple (surowe) | core |
|---|---|---|
| accel | (0, 0, +1) g | (0, 0, −9.81) m/s² |
| q | do ustalenia | (0, 1, 0, 0) — 180° wokół X |

`(0, 0, 1, 0)` (180° wokół Y) to też „ekranem w dół”, tylko z yaw różnym o 180°.

### 5.4 Stoi pionowo (portret), ekran do Ciebie, górna krawędź do sufitu

```
        +Y_B  ->  +Z_W (sufit)
          ^
          |
     .---------.
     |         |
     |  ekran  |   ⊙ +Z_B (na Ciebie)
     |         |
     '---------'
  ======= stół =======
```

| | Apple (surowe) | core |
|---|---|---|
| accel | (0, −1, 0) g | (0, +9.81, 0) m/s² |
| q | do ustalenia | (0.7071, 0.7071, 0, 0) — +90° wokół X |

Sprawdzenie: `R(q)·(0, 1, 0) = (0, 0, 1)`, górna krawędź wskazuje sufit.
Ekran (body +Z) wskazuje −Y świata.

### 5.5 Poziomo (landscape), prawa krawędź do sufitu, ekran do Ciebie

| | Apple (surowe) | core |
|---|---|---|
| accel | (−1, 0, 0) g | (+9.81, 0, 0) m/s² |
| q | do ustalenia | (0.5, 0.5, −0.5, 0.5) |

Tu obrót nie jest wokół jednej osi świata, dlatego w kwaternionie trzy składowe są niezerowe.
Sprawdzenie: `R(q)·(1, 0, 0) = (0, 0, 1)`, prawa krawędź wskazuje sufit.

### 5.6 Leży ekranem do góry i kręcisz nim w lewo 90°/s

| | Apple (surowe) | core |
|---|---|---|
| gyro | (0, 0, +1.571) rad/s | (0, 0, +1.571) rad/s |
| accel | (0, 0, −1) g | (0, 0, +9.81) m/s² |
| q po 1 s (start: 5.1) | do ustalenia | (0.7071, 0, 0, 0.7071) |

### 5.7 Leży ekranem do góry i szybko podnosisz go do góry (a = 2 m/s²)

| | Apple (surowe) | core |
|---|---|---|
| accel | ≈ (0, 0, −1.20) g | ≈ (0, 0, +11.81) m/s² |

### 5.8 Eksperyment myślowy: swobodny spadek

`f = 0`, więc accel ≈ (0, 0, 0) w obu konwencjach, w każdej orientacji.
To najlepsza ilustracja, że akcelerometr nie mierzy „grawitacji”, tylko specific force.
(Nie testujemy tego na podłodze.)

---

## 6. Testy weryfikacyjne (cube.html, etap 3)

Każdy test ma oczekiwany wynik przy obecnych hipotezach. Jeśli wynik jest inny,
poprawiamy **tylko** `frames.py` i ten dokument; nagrania zostają ważne, bo trzymają surowe dane.

| # | Test | Oczekiwanie (hipoteza) | Co rozstrzyga |
|---|---|---|---|
| V1 | Telefon płasko, ekranem do góry, nieruchomo: odczyt surowego `acc.z` i `gravity_dm.z` | ≈ −1 | znak `s` akcelerometru i grawitacji |
| V2 | Ta sama pozycja: odczyt `q_ref` | `x ≈ y ≈ 0` (sam yaw). Gdyby `|x|` lub `|y|` ≈ 1, Z Apple'a wskazywałoby w dół | Z Apple'a = w górę; zgodność osi body z rysunkiem |
| V3 | Płasko, ekranem do góry, obrót w lewo (patrząc z góry) | surowy `gyr.z > 0` | reguła prawej dłoni |
| V4 | Z pozycji płaskiej unieś górną krawędź o ~30° (obrót wokół body X) | `q_ref.x > 0` → `q_core = (w, x, y, z)`; `q_ref.x < 0` → sprzężenie | kierunek `CMQuaternion` (i Hamilton/JPL) |
| V5 | Nagranie z dowolnym ruchem, w Pythonie: porównanie `gravity_dm` z `R(q_core)ᵀ·(0, 0, −1)` | kąt między nimi < 1° przez całe nagranie | potwierdza V1 + V4 razem |
| V6 | Porównanie `t_send` z `t_dm` | `0 ≤ t_send − t_dm <` kilkadziesiąt ms | wspólna oś czasu |
| V7 | `cube.html`: kładziesz, unosisz górę, kręcisz w lewo | kostka robi dokładnie to samo; ekran kostki do góry, gdy telefon ekranem do góry | mapowanie core → three.js (sekcja 4.2) |

Po etapie 3 każdy wpis „DO WERYFIKACJI” zamienia się w „ZWERYFIKOWANE (data, test Vn)”
albo w poprawkę.

---

## 7. Czy rozumiem?

Najpierw odpowiedz sam, potem rozwiń odpowiedź.

**1. Telefon leży nieruchomo ekranem w dół. Ile wynosi `accel` w core? Co pokaże surowy
akcelerometr Apple'a, jeśli test V1 potwierdzi `s = −1`?**

<details><summary>Odpowiedź</summary>

core: `(0, 0, −9.81)` m/s². Specific force wskazuje „w górę” (stół pcha telefon),
a „w górę” to teraz −Z body, bo ekran patrzy w podłogę.
Apple: `acc = s · f / g0 = −1 · (0, 0, −1) = (0, 0, +1)` g.

</details>

**2. `q_core = (0.7071, 0, 0, 0.7071)`. Jaka to orientacja, gdzie w świecie wskazuje prawa
krawędź telefonu i co pokaże akcelerometr (telefon nieruchomy)?**

<details><summary>Odpowiedź</summary>

Obrót o 90° wokół osi Z świata (`cos 45°`, oś (0, 0, 1)) względem pozycji „płasko,
ekranem do góry, osie body = osie świata”. Prawa krawędź: `R(q)·(1, 0, 0) = (0, 1, 0)`,
czyli +Y świata. Akcelerometr: `(0, 0, +9.81)`, dokładnie tyle samo co bez obrotu,
bo obrót wokół pionu nie zmienia kierunku grawitacji w ramce body.

</details>

**3. Z telefonu przychodzi `q_ref` w kolejności Apple'a (x, y, z, w) = `(0.2588, 0, 0, 0.9659)`.
Jak wygląda w core i czego jeszcze nie wiemy?**

<details><summary>Odpowiedź</summary>

Po przestawieniu kolejności: `(0.9659, 0.2588, 0, 0)`, czyli 30° wokół osi X
(`cos 15° ≈ 0.9659`, `sin 15° ≈ 0.2588`). Nie wiemy jeszcze, czy to kwaternion
body → świat, czy odwrotny. Jeśli odwrotny, `q_core = (0.9659, −0.2588, 0, 0)`.
Rozstrzyga test V4: unosimy górną krawędź i patrzymy na znak `x`.

</details>

**4. Dlaczego w v1 główną metryką jest błąd tiltu, a nie błąd całkowity?**

<details><summary>Odpowiedź</summary>

W 6DoF jedyną informacją o „absolutnej” orientacji jest grawitacja z akcelerometru,
a ona mówi tylko, gdzie jest pion. Filtry mogą więc korygować tilt, ale yaw tylko całkują
z żyroskopu, przez co dryfuje. Referencja Apple'a w `xArbitraryZVertical` też nie używa
magnetometru, więc jej yaw też dryfuje. Błąd yaw mierzy dryf, a nie jakość filtra.

</details>

**5. Wpisujesz `q_core = (1, 0, 0, 0)` prosto w `phone.quaternion` w scenie three.js
(bez grupy `world`). Gdzie będzie patrzył ekran modelu i dlaczego to źle?**

<details><summary>Odpowiedź</summary>

Model zostanie w swoich osiach: ekran (body +Z) patrzy w +Z sceny, czyli w stronę kamery.
Telefon wygląda, jakby stał pionowo, choć naprawdę leży płasko. Powód: w three.js „w górę”
to +Y, a w core +Z. Rozwiązanie: grupa `world` obrócona o −90° wokół X
(albo `q_three = q_M ⊗ q_core`).

</details>

---

## Źródła

- Apple: [CMMotionManager](https://developer.apple.com/documentation/coremotion/cmmotionmanager) (osie urządzenia — ilustracja)
- Apple: [Getting raw accelerometer events](https://developer.apple.com/documentation/coremotion/getting-raw-accelerometer-events), [CMAcceleration](https://developer.apple.com/documentation/coremotion/cmacceleration) (jednostki g)
- Apple: [Getting raw gyroscope events](https://developer.apple.com/documentation/coremotion/getting-raw-gyroscope-events) (rad/s, bias)
- Apple: [CMDeviceMotion.gravity](https://developer.apple.com/documentation/coremotion/cmdevicemotion/gravity)
- Apple: [xArbitraryZVertical](https://developer.apple.com/documentation/coremotion/cmattitudereferenceframe/xarbitraryzvertical)
- Apple: [CMAttitude](https://developer.apple.com/documentation/coremotion/cmattitude), [CMQuaternion](https://developer.apple.com/documentation/coremotion/cmquaternion)
- Apple: [CMLogItem.timestamp](https://developer.apple.com/documentation/coremotion/cmlogitem/timestamp)
- J. Solà, *Quaternion kinematics for the error-state Kalman filter* (2017) — porównanie Hamilton vs JPL, wzory z sekcji 2.3
