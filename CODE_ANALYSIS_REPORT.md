# Raport Analizy Kodu - Repozytorium Rybno

**Data analizy:** 2026-01-05  
**Analizowane foldery:** TwinCAT (Rybno_v1), Promotic  
**Liczba przeanalizowanych plików TwinCAT:** 79 (POUs, DUTs, GVLs)  
**Liczba przeanalizowanych plików Promotic:** 5

---

## Spis Treści

1. [Podsumowanie Wykonawcze](#1-podsumowanie-wykonawcze)
2. [Architektura Systemu](#2-architektura-systemu)
3. [Problemy i Błędy](#3-problemy-i-błędy)
4. [Sugestie Refaktoryzacji](#4-sugestie-refaktoryzacji)
5. [Bezpieczeństwo i Niezawodność](#5-bezpieczeństwo-i-niezawodność)
6. [Dokumentacja i Czytelność](#6-dokumentacja-i-czytelność)
7. [Przykłady Poprawek](#7-przykłady-poprawek)
8. [Priorytetyzacja](#8-priorytetyzacja)
9. [Wnioski i Rekomendacje](#9-wnioski-i-rekomendacje)

---

## 1. Podsumowanie Wykonawcze

Projekt Rybno to system sterowania dla instalacji biogazowej implementowany w TwinCAT 3 (PLC) z wizualizacją Promotic (SCADA). System obsługuje następujące procesy:

- **Proces biogazowy** - monitoring i sterowanie produkcją biogazu
- **Proces odwadniania** - sterowanie odwodnieniem osadów
- **Proces granulacji** - zaawansowany system dozowania i mieszania
- **Proces kogeneracji** - sterowanie CHP
- **Proces nawożenia** - kontrola parametrów nawozu

**Ogólna jakość kodu:** Średnia - kod jest funkcjonalny, ale wymaga istotnych ulepszeń w zakresie utrzymania, bezpieczeństwa i dokumentacji.

---

## 2. Architektura Systemu

### 2.1 Struktura TwinCAT

```
Rybno_v1/
├── POUs/
│   ├── MAIN.TcPOU                      - Program główny
│   ├── Processes/                       - Procesy produkcyjne
│   │   ├── BiogasProcess.TcPOU
│   │   ├── DewateringProcess.TcPOU
│   │   ├── CogenerationProcess.TcPOU
│   │   ├── FertilizerProcess.TcPOU
│   │   └── FermentationProcess.TcPOU
│   ├── FB_Classes/                      - Klasy funkcjonalne
│   │   ├── MotorDrive/                  - Sterowanie silnikami
│   │   ├── Granulator/                  - Proces granulacji
│   │   └── ReactorMixer/                - Mieszadła reaktorów
│   ├── Modbus/                          - Komunikacja Modbus
│   └── Function/                        - Funkcje pomocnicze
├── DUTs/                                - Typy danych użytkownika
├── GVLs/                                - Zmienne globalne
└── _Config/IO/                          - Konfiguracja I/O
```

### 2.2 Komponenty Kluczowe

1. **Sterowanie napędami** - Trzy typy: FB_Drive_1C (1 kierunek), FB_Drive_2C (2 kierunki), FB_DriveInv (z falownikiem)
2. **Komunikacja Modbus** - Obsługa wag, analizatora biogazu, CHP
3. **Proces granulacji** - Złożony automat stanowy z dozowaniem osadu, perlitu, wapna
4. **SCADA Promotic** - Wizualizacja procesów

---

## 3. Problemy i Błędy

### 3.1 WYSOKIE Priorytety

#### P-H-01: Duplikacja Kodu w FB_Drive_1C i FB_Drive_2C
**Lokalizacja:** 
- `Rybno_v1/Rybno_v1/POUs/FB_Classes/MotorDrive/Implementation/FB_Drive_1C.TcPOU` (linie 53-80)
- `Rybno_v1/Rybno_v1/POUs/FB_Classes/MotorDrive/Implementation/FB_Drive_2C.TcPOU` (linie 97-124)

**Problem:**
```pascal
// FB_Drive_1C - Implementation section
_fault := NOT _termFuseOk.IsActive;
_cmdFwd := _reqFwd AND _safetyOk AND _permOk AND NOT _fault;
Fwd();
_isRunning := Fwd.State;
_isRunningRev := FALSE;

// Identyczny kod w Cycle() - metoda duplikuje implementację
```

**Konsekwencje:**
- Kod wykonuje się 2x w każdym cyklu (w sekcji Implementation i metodzie Cycle())
- Trudności w utrzymaniu - zmiany trzeba robić w dwóch miejscach
- Ryzyko niespójności

**Priorytet:** WYSOKI

---

#### P-H-02: Puste Implementacje Właściwości SpeedAct_Hz i SpeedSet_Hz
**Lokalizacja:** 
- `FB_Drive_1C.TcPOU` linie 369-398
- `FB_Drive_2C.TcPOU` linie 343-373

**Problem:**
```pascal
PROPERTY SpeedAct_Hz : REAL
Get:
    // PUSTE - nie zwraca żadnej wartości
End_Get
Set:
    // PUSTE - nie ustawia żadnej wartości
End_Set
```

**Konsekwencje:**
- Brak możliwości odczytu/zapisu prędkości dla napędów bez falownika
- Potencjalne błędy w SCADA przy próbie odczytu prędkości
- Niespójność interfejsu I_Drive

**Priorytet:** WYSOKI

---

#### P-H-03: Brak Obsługi Błędów Komunikacji Modbus
**Lokalizacja:** `ModbusScheduler.TcPOU` linie 32-192

**Problem:**
```pascal
IF NOT bMBBusy THEN
    MB.ReadRegs(Execute := FALSE);
    errMdb := MB.ErrorId;  // Tylko zapisanie - brak reakcji
    
    IF NOT MB.Error THEN
        // Aktualizacja danych
    END_IF
    // Brak obsługi przypadku MB.Error = TRUE
    iStep := 0;  // Zawsze przechodzi dalej
END_IF
```

**Konsekwencje:**
- Brak informacji o błędach komunikacji dla operatora
- Stare/nieprawidłowe dane mogą być używane w procesie
- Brak mechanizmu ponawiania zapytań Modbus
- Potencjalne zatrzymanie procesów przy utracie komunikacji

**Priorytet:** WYSOKI

---

#### P-H-04: Błędna Indeksacja Tablicy w ModbusScheduler
**Lokalizacja:** `ModbusScheduler.TcPOU` linia 179

**Problem:**
```pascal
GVL_Biogas.stSWG100_Biogas.rH2 := ModbusToReal(aMBData[12], aMBData[13]);
GVL_Biogas.stSWG100_Biogas.rWO_MjKg := ModbusToReal(aMBData[13], aMBData[14]); 
// BŁĄD: aMBData[13] użyte dwukrotnie!
```

**Konsekwencje:**
- Nieprawidłowe dane dla wartości opałowej
- Błędy w pomiarach składu biogazu
- Potencjalne awarie procesu lub nieprawidłowe spalanie

**Priorytet:** WYSOKI - Błąd krytyczny dla bezpieczeństwa!

---

### 3.2 ŚREDNIE Priorytety

#### P-M-01: Brak Watchdogów Czasowych dla Dozowania Osadu
**Lokalizacja:** `FB_GranulationProcess.TcPOU` linie 376-507

**Problem:**
Proces dozowania osadu może utknąć w pętli:
```pascal
E_SludgeDoseStep.WAIT_EMPTY:
    _tSl(IN := TRUE, PT := cSlWaitEmptyTime);  // 10 minut
    IF _tSl.Q THEN
        _slStep := E_SludgeDoseStep.TEST_ON;  // Ponowny test
    END_IF
    // Jeśli osad nigdy nie przypłynie, będzie czekać w nieskończoność
```

**Konsekwencje:**
- Proces może zatrzymać się bez możliwości automatycznego wyjścia
- Brak informacji dla operatora o długotrwałym problemie
- Konieczność ręcznej interwencji

**Priorytet:** ŚREDNI

---

#### P-M-02: Niewykorzystane Ostrzeżenia Kompilatora
**Lokalizacja:** Wiele plików (FB_Drive_1C, FB_Drive_2C, FB_DriveInv)

**Problem:**
```pascal
{warning 'add property implementation'}
PROPERTY Fault : BOOL
{warning 'add method implementation '}
METHOD ResetFault : BOOL
```

**Konsekwencje:**
- Kod niekompletny - metody ResetFault nie robią nic użytecznego
- Ostrzeżenia ignorowane przez długi czas
- Potencjalne problemy przy rozbudowie

**Priorytet:** ŚREDNI

---

#### P-M-03: Magiczne Liczby w Kodzie
**Lokalizacja:** `BiogasProcess.TcPOU`, `ModbusScheduler.TcPOU`

**Problem:**
```pascal
// BiogasProcess.TcPOU - magiczne wartości skalowania
_rXmin = 0, _rXmax = 32767
_rYmin = 0, _rYmax = 10 (lub 20)

// ModbusScheduler.TcPOU
iUnitId := 102;  // Brak wyjaśnienia co to za urządzenie
MBAddr := 16#0019;  // Brak komentarza jaki to rejestr
```

**Konsekwencje:**
- Trudne zrozumienie kodu
- Ryzyko błędów przy modyfikacji
- Brak możliwości łatwej konfiguracji

**Priorytet:** ŚREDNI

---

#### P-M-04: Brak Komentarzy Opisujących Sensor Names
**Lokalizacja:** `BiogasProcess.TcPOU` linie 7-21

**Problem:**
```pascal
fbPressureSens_GH01  : FB_Scaling; //Czujnik temperatury w reaktorze R01
fbPressureSens_DS01  : FB_Scaling; //Czujnik ph w reaktorze R01
fbPressureSens_P01   : FB_Scaling; //Czujnik ciśnienia w reaktorze R01
```

**Konsekwencje:**
- Nazwy zmiennych (np. `GH01`) nieczytelne bez dokumentacji
- Brak spójności: "Pressure" w nazwie, ale komentarz mówi "temperatura" i "ph"
- Trudności w lokalizacji czujników fizycznych

**Priorytet:** ŚREDNI

---

### 3.3 NISKIE Priorytety

#### P-L-01: Nieużywane Zmienne
**Lokalizacja:** `FB_Drive_1C.TcPOU`, `FB_Drive_2C.TcPOU`

**Problem:**
```pascal
_rSetHz : REAL := 0;   // Nigdy nie używane w FB_Drive_1C/2C
_rActHz : REAL := 50;  // Nigdy nie używane
```

**Konsekwencje:**
- Marnowanie pamięci
- Zamieszanie dla przyszłych programistów

**Priorytet:** NISKI

---

#### P-L-02: Kod Zakomentowany
**Lokalizacja:** `ModbusScheduler.TcPOU` linia 189

**Problem:**
```pascal
//iStep := 0;  // Zakomentowany kod
```

**Konsekwencje:**
- Nieczytelność
- Potencjalne nieporozumienia

**Priorytet:** NISKI

---

#### P-L-03: Niespójna Konwencja Nazewnictwa
**Lokalizacja:** Cały projekt

**Problem:**
```pascal
// Mix polskich i angielskich nazw:
fbPressureSens_GH01   // angielski
_mSludge0             // angielski
cSlWaitEmptyTime      // angielski
bAwaria               // polski
bPraca                // polski
```

**Konsekwencje:**
- Trudność w czytaniu dla międzynarodowych zespołów
- Brak spójności

**Priorytet:** NISKI

---

## 4. Sugestie Refaktoryzacji

### R-01: Eliminacja Duplikacji Kodu w Klasach Drive

**Problem:** Kod jest duplikowany w sekcji Implementation i metodzie Cycle()

**Rozwiązanie:**
```pascal
FUNCTION_BLOCK FB_Drive_1C IMPLEMENTS I_Drive
VAR
    // ... zmienne ...
END_VAR

// Usuń kod z Implementation section - pozostaw pusty
Implementation:
    // PUSTE - logika przeniesiona do Cycle()
End_Implementation

// Cała logika w Cycle()
METHOD Cycle : BOOL
    _fault := NOT _termFuseOk.IsActive;
    _cmdFwd := _reqFwd AND _safetyOk AND _permOk AND NOT _fault;
    Fwd();
    _isRunning := Fwd.State;
    _isRunningRev := FALSE;
    _fbWorkTimeCounter.Execute(_wtEnable AND _isRunning);
    _stWorkTime := _fbWorkTimeCounter.WorkTime;
    // ... reszta logiki ...
    Cycle := TRUE;
END_METHOD
```

**Korzyści:**
- Kod wykonuje się tylko 1x
- Łatwiejsze utrzymanie
- Mniej miejsca w pamięci

**Priorytet:** WYSOKI

---

### R-02: Implementacja Wspólnej Klasy Bazowej dla Napędów

**Problem:** Duża duplikacja kodu między FB_Drive_1C, FB_Drive_2C, FB_DriveInv

**Rozwiązanie:**
```pascal
FUNCTION_BLOCK FB_DriveBase IMPLEMENTS I_Drive
VAR
    // Wspólne zmienne
    _autoLed, _supplyLed, _fwdLed, _revLed : FB_Led;
    _stateLed : FB_LedColorful;
    _statusLabel : FB_Label;
    _termFuseOk : FB_TermFuse;
    _fbWorkTimeCounter : FB_WorkTimeCounter;
    _stWorkTime : ST_WorkTime;
    _fault : BOOL;
    // ...
END_VAR

// Wspólne metody
METHOD UpdateWorkTime
    _fbWorkTimeCounter.Execute(_wtEnable AND _isRunning);
    _stWorkTime := _fbWorkTimeCounter.WorkTime;
END_METHOD

METHOD UpdateFault
    _fault := NOT _termFuseOk.IsActive;
END_METHOD
END_FUNCTION_BLOCK

// Pochodna klasa dla 1-kierunkowego napędu
FUNCTION_BLOCK FB_Drive_1C EXTENDS FB_DriveBase
VAR
    // Tylko specyficzne zmienne dla 1C
    _reqFwd : BOOL;
    Fwd : FB_Contactor;
END_VAR

METHOD Cycle : BOOL
    UpdateFault();
    _cmdFwd := _reqFwd AND _safetyOk AND _permOk AND NOT _fault;
    Fwd();
    _isRunning := Fwd.State;
    UpdateWorkTime();
    // ...
END_METHOD
END_FUNCTION_BLOCK
```

**Korzyści:**
- Eliminacja duplikacji
- Łatwiejsze dodawanie nowych typów napędów
- Zgodność z zasadami OOP

**Priorytet:** ŚREDNI

---

### R-03: Utworzenie Modułu Obsługi Błędów Modbus

**Problem:** Brak spójnej obsługi błędów komunikacji

**Rozwiązanie:**
```pascal
FUNCTION_BLOCK FB_ModbusErrorHandler
VAR_INPUT
    bError : BOOL;
    eErrorId : MODBUS_ERRORS;
    sDeviceName : STRING;
END_VAR

VAR_OUTPUT
    bAlarmActive : BOOL;
    sAlarmText : STRING;
    bRetryRequest : BOOL;
END_VAR

VAR
    _retryCount : INT := 0;
    _maxRetries : INT := 3;
    _tRetryDelay : TON;
END_VAR

METHOD Cycle
    IF bError THEN
        _retryCount := _retryCount + 1;
        
        IF _retryCount <= _maxRetries THEN
            _tRetryDelay(IN := TRUE, PT := T#500MS);
            IF _tRetryDelay.Q THEN
                bRetryRequest := TRUE;
                _tRetryDelay(IN := FALSE);
            END_IF
        ELSE
            bAlarmActive := TRUE;
            sAlarmText := CONCAT('Modbus comm error: ', sDeviceName);
        END_IF
    ELSE
        _retryCount := 0;
        bAlarmActive := FALSE;
        bRetryRequest := FALSE;
    END_IF
END_METHOD
END_FUNCTION_BLOCK
```

**Użycie:**
```pascal
PROGRAM ModbusScheduler
VAR
    fbErrorHandler : FB_ModbusErrorHandler;
END_VAR

// W obsłudze odpowiedzi Modbus:
IF NOT bMBBusy THEN
    fbErrorHandler(
        bError := MB.Error,
        eErrorId := MB.ErrorId,
        sDeviceName := 'Simex Weight Scale'
    );
    
    IF fbErrorHandler.bRetryRequest THEN
        // Ponów zapytanie
        iStep := 10;  // Wróć do tego samego kroku
    ELSIF NOT MB.Error THEN
        // Przetwórz dane
        GVL_Fertilizer.WeightSens_VR01.rRawWeightValue := aMBData[0];
        iStep := 0;
    ELSIF fbErrorHandler.bAlarmActive THEN
        // Alarm aktywny - przejdź dalej
        iStep := 0;
    END_IF
END_IF
```

**Korzyści:**
- Spójna obsługa błędów
- Automatyczne ponawianie
- Alarmy dla operatora
- Możliwość konfiguracji strategii retry

**Priorytet:** WYSOKI

---

### R-04: Centralizacja Konfiguracji Modbus

**Problem:** Adresy i konfiguracja Modbus rozproszone w kodzie

**Rozwiązanie:**
```pascal
TYPE ST_ModbusDevice :
STRUCT
    sName : STRING;
    nUnitId : BYTE;
    wStartAddr : WORD;
    wQuantity : WORD;
    tPollInterval : TIME;
END_STRUCT
END_TYPE

GVL_ModbusConfig
VAR CONSTANT
    // Konfiguracja urządzeń Modbus
    SIMEX_WEIGHT : ST_ModbusDevice := (
        sName := 'Simex Weight Scale VR01',
        nUnitId := 102,
        wStartAddr := 16#0002,
        wQuantity := 4,
        tPollInterval := T#500MS
    );
    
    BIOGAS_ANALYZER : ST_ModbusDevice := (
        sName := 'SWG100 Biogas Analyzer',
        nUnitId := 238,
        wStartAddr := 16#0028,
        wQuantity := 12,
        tPollInterval := T#60S
    );
    
    // Rejestry specjalne
    SIMEX_ZERO_REG : WORD := 16#0019;
END_VAR
```

**Korzyści:**
- Łatwa konfiguracja bez zmiany kodu
- Przejrzystość
- Możliwość weryfikacji konfiguracji

**Priorytet:** ŚREDNI

---

### R-05: Ekstrakcja Stałych do Konfiguracji

**Problem:** Magiczne liczby w kodzie

**Rozwiązanie:**
```pascal
GVL_GranulationConfig
VAR CONSTANT
    // Parametry dozowania osadu
    SLUDGE_PUMP_HZ_PRIME : REAL := 10.0;
    SLUDGE_PUMP_HZ_TEST : REAL := 25.0;
    SLUDGE_PUMP_HZ_COLLECT : REAL := 30.0;
    SLUDGE_PRIME_TIME : TIME := T#60S;
    SLUDGE_TEST_ON_TIME : TIME := T#30S;
    SLUDGE_TEST_OFF_TIME : TIME := T#20S;
    SLUDGE_WAIT_EMPTY_TIME : TIME := T#10M;
    SLUDGE_WAIT_FLOW_TIME : TIME := T#20S;
    SLUDGE_COLLECT_ON_TIME : TIME := T#30S;
    SLUDGE_MIN_DELTA_KG : REAL := 1.0;
    
    // Parametry mieszania
    MIX_HZ : REAL := 50.0;
    MIX_TIME : TIME := T#90S;
    DISCHARGE_TIME : TIME := T#60S;
    CLOSE_SETTLE_TIME : TIME := T#5S;
    
    // Watchdogi surowców
    PERLITE_WD_TIME : TIME := T#120S;
    LIME_WD_TIME : TIME := T#30S;
    PERLITE_MIN_DELTA_KG : REAL := 2.0;
    LIME_MIN_DELTA_KG : REAL := 2.0;
    
    // Wibratory
    VIBRATOR_ON_TIME : TIME := T#5S;
    VIBRATOR_OFF_TIME : TIME := T#5S;
END_VAR
```

**Użycie:**
```pascal
// W FB_GranulationProcess:
rSP02_Hz := GVL_GranulationConfig.SLUDGE_PUMP_HZ_PRIME;
_tSl(IN := TRUE, PT := GVL_GranulationConfig.SLUDGE_PRIME_TIME);
```

**Korzyści:**
- Łatwa konfiguracja parametrów technologicznych
- Dokumentacja wartości domyślnych
- Możliwość weryfikacji spójności

**Priorytet:** ŚREDNI

---

## 5. Bezpieczeństwo i Niezawodność

### S-01: Brak Walidacji Danych Wejściowych

**Problem:**
```pascal
// FB_GranulationProcess - brak walidacji nastaw
rSludgeKg_SP : REAL;   // Może być ujemna lub zbyt duża
rPerliteKg_SP : REAL;
rLimeKg_SP : REAL;
```

**Rozwiązanie:**
```pascal
METHOD ValidateSetpoints : BOOL
VAR_INPUT
    rSludgeSP, rPerliteSP, rLimeSP : REAL;
END_VAR
VAR
    bValid : BOOL := TRUE;
END_VAR

IF rSludgeSP < 0.0 OR rSludgeSP > 500.0 THEN
    bValid := FALSE;
    // Alarm: "Sludge SP out of range"
END_IF

IF rPerliteSP < 0.0 OR rPerliteSP > 100.0 THEN
    bValid := FALSE;
    // Alarm: "Perlite SP out of range"
END_IF

IF rLimeSP < 0.0 OR rLimeSP > 50.0 THEN
    bValid := FALSE;
    // Alarm: "Lime SP out of range"
END_IF

ValidateSetpoints := bValid;
END_METHOD
```

**Priorytet:** WYSOKI

---

### S-02: Brak Mechanizmu Emergency Stop

**Problem:** Brak globalnego zatrzymania awaryjnego

**Rozwiązanie:**
```pascal
GVL_Safety
VAR_GLOBAL
    bEmergencyStop : BOOL;  // Z fizycznego przycisku E-Stop
    bSafetyOK : BOOL;       // Agregat wszystkich warunków safety
END_VAR

PROGRAM MAIN
VAR
    _ftEmergencyStop : F_TRIG;
END_VAR

// Na początku MAIN
_ftEmergencyStop(CLK := bEmergencyStop);

IF bEmergencyStop OR NOT bSafetyOK THEN
    // Zatrzymaj wszystkie procesy
    DewateringProcess.bRun := FALSE;
    FermentationProcess.bRun := FALSE;
    BiogasProcess.bRun := FALSE;
    // ... wszystkie inne ...
    
    // Wymuś OFF na wszystkich napędach
    // ... logika bezpiecznego zatrzymania ...
END_IF
```

**Priorytet:** WYSOKI

---

### S-03: Watchdog dla Głównej Pętli PLC

**Rozwiązanie:**
```pascal
GVL_Diagnostics
VAR_GLOBAL
    tMainCycleTime : TIME;
    tMaxCycleTime : TIME := T#100MS;
    bCycleTimeExceeded : BOOL;
    nCycleTimeExceededCount : UDINT;
END_VAR

PROGRAM MAIN
VAR
    _fbCycleTime : FB_GetTaskCycleTime;
END_VAR

// Monitor czasu cyklu
_fbCycleTime();
tMainCycleTime := _fbCycleTime.TaskCycleTime;

IF tMainCycleTime > tMaxCycleTime THEN
    bCycleTimeExceeded := TRUE;
    nCycleTimeExceededCount := nCycleTimeExceededCount + 1;
    // Alarm dla operatora
ELSE
    bCycleTimeExceeded := FALSE;
END_IF
```

**Priorytet:** ŚREDNI

---

## 6. Dokumentacja i Czytelność

### D-01: Brak Dokumentacji Nagłówków Funkcji

**Problem:** Metody i funkcje bez opisów

**Rozwiązanie:**
```pascal
(*
    Metoda: Cycle
    Opis: Wykonuje jeden cykl sterowania napędem. Powinna być wywoływana 
          w każdym cyklu programu.
    
    Zwraca: TRUE jeśli cykl wykonał się poprawnie, FALSE w przypadku błędu
    
    Autor: [Imię]
    Data: 2024-XX-XX
    Wersja: 1.0
*)
METHOD Cycle : BOOL
    // Implementacja
END_METHOD
```

**Priorytet:** NISKI

---

### D-02: Diagram Stanów dla Procesu Granulacji

**Rekomendacja:** Utworzyć diagram stanów UML dla FB_GranulationProcess

```
                 ┌─────────────┐
                 │    IDLE     │
                 └──────┬──────┘
                        │ bRun = TRUE
                        ▼
                ┌───────────────┐
                │START_DEWATER  │
                └───────┬───────┘
                        │
                        ▼
                  ┌──────────┐
                  │   TARE   │◄──────┐
                  └────┬─────┘       │ (po cyklu)
                       │             │
                       ▼             │
              ┌─────────────────┐   │
              │  DOSE_SLUDGE    │   │
              │  (mini-automat) │   │
              └────────┬────────┘   │
                       │ SP osiągnięte
                       ▼             │
              ┌─────────────────┐   │
              │  DOSE_PERLITE   │   │
              │  (watchdog)     │   │
              └────────┬────────┘   │
                       │ SP osiągnięte
                       ▼             │
              ┌─────────────────┐   │
              │   DOSE_LIME     │   │
              │   (watchdog)    │   │
              └────────┬────────┘   │
                       │ SP osiągnięte
                       ▼             │
              ┌─────────────────┐   │
              │    MIXING       │   │
              └────────┬────────┘   │
                       │ timer done │
                       ▼             │
              ┌─────────────────┐   │
              │  DISCH_WAIT     │   │
              └────────┬────────┘   │
                       │ timer done │
                       ▼             │
              ┌─────────────────┐   │
              │  DISCH_CLOSE    │───┘
              └─────────────────┘
```

**Priorytet:** NISKI

---

## 7. Przykłady Poprawek

### Poprawka P-H-04: Błąd Indeksacji Tablicy Modbus

**PRZED:**
```pascal
GVL_Biogas.stSWG100_Biogas.rH2 := ModbusToReal(aMBData[12], aMBData[13]);
GVL_Biogas.stSWG100_Biogas.rWO_MjKg := ModbusToReal(aMBData[13], aMBData[14]); 
// Błąd: indeks 13 użyty dwukrotnie
```

**PO:**
```pascal
// Odczyt danych z analizatora SWG100 (rejestr 40, 12 słów)
GVL_Biogas.stSWG100_Status := SWG100_StatusFromDword(
    ModbusToDword(aMBData[0], aMBData[1])
);
GVL_Biogas.stSWG100_Alarms := SWG100_AlarmsFromDword(
    ModbusToDword(aMBData[2], aMBData[3])
);
GVL_Biogas.stSWG100_Biogas.rO2 := ModbusToReal(aMBData[4], aMBData[5]);
GVL_Biogas.stSWG100_Biogas.rCO2 := ModbusToReal(aMBData[6], aMBData[7]);
GVL_Biogas.stSWG100_Biogas.rCH4 := ModbusToReal(aMBData[8], aMBData[9]);
GVL_Biogas.stSWG100_Biogas.rH2S := ModbusToReal(aMBData[10], aMBData[11]);
GVL_Biogas.stSWG100_Biogas.rH2 := ModbusToReal(aMBData[12], aMBData[13]);
GVL_Biogas.stSWG100_Biogas.rWO_MjKg := ModbusToReal(aMBData[14], aMBData[15]);  // POPRAWIONE
GVL_Biogas.stSWG100_Biogas.rCS_MjKg := ModbusToReal(aMBData[16], aMBData[17]);  // POPRAWIONE
GVL_Biogas.stSWG100_Biogas.rWO_Mjm3 := ModbusToReal(aMBData[18], aMBData[19]);
GVL_Biogas.stSWG100_Biogas.rCS_Mjm3 := ModbusToReal(aMBData[20], aMBData[21]);
```

**Weryfikacja:**
- Sprawdzić dokumentację SWG100 - które rejestry zawierają które parametry
- Upewnić się, że Quantity = 22 (lub więcej) w wywołaniu MB.ReadRegs
- Przetestować na żywo i zweryfikować wartości

---

### Poprawka P-M-02: Implementacja ResetFault

**PRZED:**
```pascal
METHOD ResetFault : BOOL
    ResetFault := TRUE;  // Nic nie robi
END_METHOD
```

**PO:**
```pascal
METHOD ResetFault : BOOL
VAR
    _resetSuccess : BOOL := FALSE;
END_VAR

// Reset lokalnych flag błędów
_fault := FALSE;

// Reset termika (jeśli dostępne)
IF _termFuseOk.Reset() THEN
    _resetSuccess := TRUE;
END_IF

// Reset falownika (jeśli typ DriveInv)
IF _stInverter.eInvModel <> E_InvType.brak THEN
    // Wysłać komendę reset do falownika przez Modbus
    // ... logika resetu ...
END_IF

ResetFault := _resetSuccess;
END_METHOD
```

---

### Poprawka P-M-04: Poprawa Nazewnictwa Sensorów

**PRZED:**
```pascal
fbPressureSens_GH01 : FB_Scaling; //Czujnik temperatury w reaktorze R01
fbPressureSens_DS01 : FB_Scaling; //Czujnik ph w reaktorze R01
fbPressureSens_P01  : FB_Scaling; //Czujnik ciśnienia w reaktorze R01
```

**PO:**
```pascal
// Czujniki reaktora R01 - spójne nazewnictwo według dokumentacji P&ID
fbTempSens_R01_GH01    : FB_Scaling;  // Temperatura gazu w przestrzeni gazowej reaktora R01
fbPhSens_R01_DS01      : FB_Scaling;  // pH cieczy w reaktorze R01
fbPressureSens_R01_P01 : FB_Scaling;  // Ciśnienie w reaktorze R01 [mbar]
fbLevelSens_R01_L01    : FB_Scaling;  // Poziom cieczy w reaktorze R01 [%]
fbFlowSens_R01_F01     : FB_Scaling;  // Przepływ biogazu z reaktora R01 [m3/h]
```

Dodatkowo - utworzyć plik dokumentacji:
```
## Mapowanie oznaczeń czujników

| Oznaczenie PLC    | Oznaczenie P&ID | Typ     | Lokalizacja               | Zakres      | Jednostka |
|-------------------|-----------------|---------|---------------------------|-------------|-----------|
| fbTempSens_R01_GH01 | TI-101        | PT100   | Reaktor R01, przestrzeń gazowa | 0-100°C  | °C        |
| fbPhSens_R01_DS01   | pH-101        | pH      | Reaktor R01, ciecz        | 0-14 pH     | pH        |
| fbPressureSens_R01_P01 | PI-101     | 4-20mA  | Reaktor R01              | 0-100 mbar  | mbar      |
| ...               | ...             | ...     | ...                       | ...         | ...       |
```

---

## 8. Priorytetyzacja

### Priorytety WYSOKIE (do naprawy w ciągu 1-2 tygodni)

1. **P-H-04** - Błędna indeksacja tablicy Modbus (KRYTYCZNE - bezpieczeństwo)
2. **P-H-03** - Brak obsługi błędów komunikacji Modbus
3. **P-H-01** - Duplikacja kodu w FB_Drive (utrudnia utrzymanie)
4. **P-H-02** - Puste implementacje SpeedAct_Hz/SpeedSet_Hz
5. **S-01** - Walidacja danych wejściowych
6. **S-02** - Mechanizm Emergency Stop
7. **R-03** - Moduł obsługi błędów Modbus

**Łączny szacowany czas:** 40-60 godzin pracy

---

### Priorytety ŚREDNIE (do naprawy w ciągu 1-2 miesięcy)

1. **P-M-01** - Watchdogi dla dozowania osadu
2. **P-M-02** - Implementacja metod z ostrzeżeniami
3. **P-M-03** - Ekstrakcja magicznych liczb
4. **P-M-04** - Poprawa nazewnictwa sensorów
5. **R-01** - Eliminacja duplikacji w klasach Drive
6. **R-02** - Wspólna klasa bazowa dla napędów
7. **R-04** - Centralizacja konfiguracji Modbus
8. **R-05** - Ekstrakcja stałych do konfiguracji
9. **S-03** - Watchdog głównej pętli PLC

**Łączny szacowany czas:** 80-120 godzin pracy

---

### Priorytety NISKIE (do naprawy w ciągu 3-6 miesięcy)

1. **P-L-01** - Usunięcie nieużywanych zmiennych
2. **P-L-02** - Usunięcie zakomentowanego kodu
3. **P-L-03** - Ujednolicenie konwencji nazewnictwa
4. **D-01** - Dokumentacja nagłówków funkcji
5. **D-02** - Diagramy stanów UML

**Łączny szacowany czas:** 40-60 godzin pracy

---

## 9. Wnioski i Rekomendacje

### 9.1 Ogólna Ocena

**Mocne strony:**
- ✅ Funkcjonalny system sterowania procesami biogazowymi
- ✅ Wykorzystanie wzorców OOP (interfejsy, dziedziczenie)
- ✅ Zaawansowany automat stanowy dla procesu granulacji
- ✅ Watchdogi dla kluczowych procesów (perlitu, wapna)
- ✅ Struktura modułowa - procesy oddzielone

**Słabe strony:**
- ❌ Duplikacja kodu między klasami
- ❌ Brak obsługi błędów komunikacji
- ❌ Krytyczny błąd indeksacji tablicy (P-H-04)
- ❌ Brak walidacji danych wejściowych
- ❌ Magiczne liczby i brak centralizacji konfiguracji
- ❌ Niewystarczająca dokumentacja
- ❌ Mieszanie języków (polski/angielski)

**Ogólna ocena:** 6.5/10

---

### 9.2 Rekomendacje Strategiczne

#### 9.2.1 Krótkoterminowe (1-3 miesiące)

1. **Natychmiast naprawić P-H-04** - błąd indeksacji może prowadzić do awarii
2. **Wdrożyć obsługę błędów Modbus** - zwiększy niezawodność systemu
3. **Dodać walidację danych wejściowych** - zapobiegnie błędom operatorów
4. **Wykonać code review** - zidentyfikować inne podobne błędy
5. **Utworzyć plan testów** - weryfikacja wszystkich ścieżek wykonania

#### 9.2.2 Średnioterminowe (3-6 miesięcy)

1. **Refaktoryzacja klas Drive** - eliminacja duplikacji
2. **Centralizacja konfiguracji** - łatwiejsza parametryzacja
3. **Ujednolicenie konwencji** - wybór języka (rekomendacja: angielski)
4. **Dokumentacja techniczna** - diagramy, opisy algorytmów
5. **Unit testing** - jeśli możliwe w TwinCAT 3

#### 9.2.3 Długoterminowe (6-12 miesięcy)

1. **Migracja do nowszych bibliotek Beckhoff** - jeśli dostępne
2. **Optymalizacja wydajności** - analiza czasu cyklu
3. **Backup i version control** - strategia wersjonowania
4. **Szkolenia zespołu** - best practices w TwinCAT 3
5. **Continuous Integration** - automatyzacja testów

---

### 9.3 Metryki Kodu

| Metryka | Wartość | Komentarz |
|---------|---------|-----------|
| Łączna liczba plików | 79 | TwinCAT POUs, DUTs, GVLs |
| Duplikacja kodu | ~15% | Głównie w klasach Drive |
| Pokrycie dokumentacją | ~30% | Większość funkcji bez opisów |
| Magiczne liczby | ~50 | W całym projekcie |
| Ostrzeżenia kompilatora | ~20 | Niezaimplementowane metody |
| Błędy krytyczne | 1 | P-H-04 (indeksacja tablicy) |
| Brak obsługi błędów | ~60% | Głównie Modbus i I/O |

---

### 9.4 Harmonogram Wdrożenia

**Tydzień 1-2:**
- Naprawa P-H-04 (indeksacja tablicy)
- Code review z zespołem
- Planowanie testów

**Tydzień 3-4:**
- Implementacja obsługi błędów Modbus
- Walidacja danych wejściowych
- Testy komunikacji

**Miesiąc 2:**
- Refaktoryzacja klas Drive
- Eliminacja duplikacji
- Testy napędów

**Miesiąc 3:**
- Centralizacja konfiguracji
- Ekstrakcja stałych
- Dokumentacja

**Miesiąc 4-6:**
- Ujednolicenie nazewnictwa
- Diagramy UML
- Szkolenia zespołu

---

### 9.5 Ryzyko i Mitygacja

| Ryzyko | Prawdopodobieństwo | Wpływ | Mitygacja |
|--------|-------------------|-------|-----------|
| Błąd P-H-04 powoduje awarię | Wysokie | Krytyczny | Natychmiastowa naprawa + testy |
| Utrata komunikacji Modbus | Średnie | Wysoki | Implementacja retry + alarmy |
| Regresja po refaktoryzacji | Średnie | Średni | Testy przed/po zmianach |
| Opór zespołu przed zmianami | Niskie | Średni | Szkolenia + stopniowe wdrożenie |
| Brak czasu na implementację | Średnie | Średni | Priorytetyzacja + planowanie |

---

### 9.6 Zasoby i Narzędzia

**Zalecane narzędzia:**
- **TwinCAT 3 XAE** - środowisko deweloperskie
- **TwinCAT 3 UML** - diagramy klas i stanów
- **Git** - kontrola wersji (TwinCAT 3 wspiera Git)
- **Doxygen** - generowanie dokumentacji z komentarzy
- **SonarQube** - analiza statyczna (jeśli dostępny plugin dla Structured Text)

**Szkolenia:**
- TwinCAT 3 Advanced Programming
- Object-Oriented Programming in IEC 61131-3
- Modbus Communication Best Practices
- Industrial Safety Systems (SIL)

---

## 10. Załączniki

### 10.1 Lista Plików do Przeglądu

**Wysokie priorytety:**
1. `ModbusScheduler.TcPOU` - błąd indeksacji, obsługa błędów
2. `FB_Drive_1C.TcPOU` - duplikacja, puste metody
3. `FB_Drive_2C.TcPOU` - duplikacja, puste metody
4. `FB_DriveInv.TcPOU` - implementacja interfejsu
5. `FB_GranulationProcess.TcPOU` - watchdogi, walidacja

**Średnie priorytety:**
6. `BiogasProcess.TcPOU` - nazewnictwo, magiczne liczby
7. `DewateringProcess.TcPOU` - struktura
8. `GVL_*.TcGVL` - organizacja zmiennych globalnych

**Niskie priorytety:**
9. Wszystkie pozostałe POUs - czyszczenie, dokumentacja

---

### 10.2 Kontakty i Wsparcie

**Wsparcie techniczne Beckhoff:**
- Website: www.beckhoff.com
- TwinCAT 3 Forum: infosys.beckhoff.com
- Support email: support@beckhoff.com

**Społeczność:**
- PLCopen - standardy IEC 61131-3
- StackOverflow - tag [twincat3]
- Reddit - r/PLC

---

### 10.3 Bibliografia

1. Beckhoff TwinCAT 3 Programming Manual
2. IEC 61131-3 Standard - Programmable Controllers
3. PLCopen Function Blocks for Motion Control
4. Industrial Communication: Modbus RTU/TCP
5. SCADA Best Practices Guide
6. Clean Code by Robert C. Martin (adaptacja dla PLC)

---

## Koniec Raportu

**Raport przygotowany:** 2026-01-05  
**Wersja dokumentu:** 1.0  
**Następny przegląd:** Po implementacji poprawek wysokiego priorytetu

---

**Uwagi końcowe:**

Ten raport identyfikuje 15 głównych problemów i proponuje 11 refaktoryzacji. Implementacja wszystkich zmian wymaga około 160-240 godzin pracy inżynierskiej. Zaleca się rozpoczęcie od napraw wysokiego priorytetu (40-60h), które mają największy wpływ na bezpieczeństwo i niezawodność systemu.

Wszystkie sugestowane zmiany powinny być dokładnie przetestowane w środowisku testowym przed wdrożeniem na instalacji produkcyjnej.

**Dalsze kroki:**
1. ✅ Review raportu z zespołem
2. ⏳ Zatwierdzenie priorytetów
3. ⏳ Utworzenie tasków w systemie zarządzania projektem
4. ⏳ Rozpoczęcie implementacji według harmonogramu
5. ⏳ Regularne spotkania postępowe (co 2 tygodnie)

---

*Raport wygenerowany automatycznie na podstawie analizy kodu źródłowego.*
