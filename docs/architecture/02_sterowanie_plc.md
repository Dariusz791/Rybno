# 02 - Sterowanie PLC (TwinCAT)

Program PLC przygotowano głownie z myślą o sterowaniu napędami (mieszadła, pompy) oraz pochodnią biogazu. Sterowanie realizowane jest przez programy procesowe
, które implementują logikę operacyjną i interfejsy do SCADA. Program przygotowano głównie w języku Structured Text (ST) z wykorzystaniem funkcji bloków (FB) i struktur danych (GVL). Część logiki sterowania napędami została zaimplementowana w sposób obiektowy, z wykorzystaniem wzorca Bridge do oddzielenia warstwy sterowania od warstwy sprzętowej (styczniki, invertery).

## Cel dokumentu

Ten dokument opisuje implementacje sterowania w PLC: programy, GVL, struktury danych i zaleznosci.

## Programy procesowe

Programy procesowe implementują logikę sterowania dla poszczególnych obiektów technologicznych i grup urządzeń. Przykładowe programy:

- FermentationProcess
- BiogasProcess
- CogenerationProcess
- FertilizerProcess
- DewateringProcess
- GranulationProcess

## Kluczowe GVL

GVL (Global Variable List) zawierają struktury danych i zmienne globalne używane w programach procesowych. Przykładowe GVL:

- GVL_Fermentation
- GVL_Biogas
- GVL_Cogeneration
- GVL_Fertilizer
- GVL_Dewatering
Struktury te są używane do przechowywania statusów, setpointów, alarmów i innych danych procesowych, które są mapowane do SCADA za pomocą OPC UA. Dane te są również wykorzystywane w logice sterowania do podejmowania decyzji i realizacji funkcji procesowych.

## Dewatering (SPD01)

### Program DewateringProcess

Do obsługi stacji odwodnienia zaimplementowano PRG DewateringProcess, który realizuje logikę sterowania podrzędnym systemem autonomicznym odwadniania osadów na prasie pierścieniowej wraz z stacja dozowania polielektrolitu SPD01. System ten jest załączany przyciskiem START z SCADA i PLC ustawia wyjście przekaźnika sterującego za pomocą bloku funkcyjnego fbContactStart typu NO. Zwrotnie PLC odbiera sygnały z wejść dwustanowych, które informuja o pracy i awarii fbContactPraca typu NO i fbContactAwaria typu NC. Czyli zasada pracy jest następująca zdalny start ze SCADA i odbieramy informacje o pracy lub awarii ze stacji odwadniania osadu. Owdodnione osady z prasy trafiają do kosza zasypowego pompy SP02, której zadaniem jest transport odwodnionego osadu lub pofermentu do reaktora fermentacji R01 lub reaktora waloryzacji i granulacji VR01.

### Napedy

W stacji odwadniania nie ma napędów sterowanych z podprogramu Dewatering. Stacja odwadniania ma swój autonomiczny system z pompą SP01, która jest sterowana z autonomicznego systemu.

## Fermentacja (R01)

### Program FermentationProcess

Do obsługi stacji fermentacji zaimplementowano PRG FermentationProcess, który realizuje logikę sterowania głównym reaktorem fermentacji R01. Program ten zarządza procesem fermentacji, monitoruje kluczowe parametry (temperatura, pH, ciśnienie, poziom) i steruje urządzeniami takimi jak mieszadła. Napędy mieszadeł są sterowane za pomocą funkcji fbDrive_MX01 i fbDrive_MX02, które są dedykowane dla napedów ze stycznikami i obsługiwane poprzez Fb_Drive_1C. Mieszadła mają nadrzędny function block fbReactorMixers, który zarządza logiką sterowania mieszadłami. Logika polega na tym, że mieszadła pracują w trybie cyklicznym gdzie każde z mieszadeł jest załączane na czas Ton np. 15 minut, a następnie wyłączane na czas Toff np. 15 minut. Ten cykl jest powtarzany tak długo jak reaktor R01 jest w trybie pracy automatycznej i nie występują alarmy krytyczne. 

### Napedy i mieszanie

- fbDrv_MX01 - oop **obiekt** klasy Drive dla mieszadła MX01, sterowany przez funkcję FB_Drive_1C();
- fbDrv_MX02 - oop **obiekt** klasy Drive dla mieszadła MX02, sterowany przez funkcję FB_Drive_1C();
- fbReactorMixers - **blok funkcyjny** FB_ReactorMixers, który zarządza logiką sterowania mieszadłami w reaktorze R01. Ten function block implementuje logikę cyklicznego załączania i wyłączania mieszadeł na podstawie czasu Ton i Toff oraz stanu pracy reaktora.

### GVL fermentacji

- **GVL**: GVL_Fermentation.stDriveStatus_MX01
- **GVL**: GVL_Fermentation.stDriveStatus_MX02
- **GVL**: GVL_Fermentation.stMixStation

### Sensory

- TempSens_R01
- PhSens_R01
- PressureSens_R01
- HydroPressSens_R01
- LevelSens_R01

## Biogaz

### Program BiogasProcess

Do obłsugi węzła biogazu zaimplementowani PRG BiogasProcess, który realizuje logikę sterowania ścieżką biogazu od reaktora do kogeneratora CHP01. Realizuje również wymiane danych z po0chodnią biogazu (TCH01) oraz analizatorem gazu (SWG100). Pochodnia biogazu posiada własny autonomiczny system sterowania  oparty na sterowniku CX7080, który komunikuje się z głównym PLC przez ADS. Dane z analizatora gazu SWG100 są odczytywane przez Modbus RTU i mapowane do struktur danych w GVL_Biogas. Dane te są wykorzystywane do monitorowania jakości biogazu (CH4, CO2, H2S, O2).

### Sensory i analiza

Węzeł biogazu jest wyposażony w następujące sensory i urządzenia pomiarowe:

- Czujnik ciśnienia PressureSens_GH01
- FlowSens_F01
- Analizator gazu SWG100 (CH4, CO2, H2S, O2, Wartość opałowa, Ciepło spalania w MJ/m3 i MJ/kg). Dane z analizatora są odczytywane przez Modbus RTU i mapowane do struktur danych w GVL_Biogas:
  - stSWG100_Biogas: struktura danych odczytanych z analizatora gazu SWG100 (CH4, CO2, H2S, O2, Wartość opałowa, Ciepło spalania w MJ/m3 i MJ/kg)
  - stSWG100_Status: struktura statusowa analizatora (komunikacja, alarmy, stan pracy)
  - stSWG100_Alarms: struktura alarmowa analizatora (aktywny alarm, typ alarmu, opis)

### Funkcje odeczytująca SWG100 po Modbus RTU

Do odczytu danych z analizatora SWG100 wykorzystujemy protokół Modbus RTU. Implementacja obejmuje funkcję odczytującą rejestry Modbus i mapującą je na strukturę stSWG100_Biogas oraz stSWG100_Status oraz stSWG100_Alarms. Zaimplementowano PRG ModbusScheduler, który cyklicznie odczytuje dane z analizatora i aktualizuje odpowiednie struktury danych w GVL. Ten sam PRG obługuuje rónież odczyt danych z wagi SIMEX z przetwornika SWI-940, która jest wykorzystywana do ważenia osadów odwopdnionych które trafiaja do granulacji. Dane z wagi są mapowane na strukturę stSIMEX_Weight w GVL_Dewatering. Oraz będą wykorzystane do odczytu danych z CHPO dachs. W tej chwili jeszcze nie uruchomione - do zrobienia.

## Nawozy

### Program FertilizerProcess

Do obsługi węzła nawozów zaimplementowano PRG FertilizerProcess, który realizuje logikę sterowania procesem produkcji nawozów z pofermentu. Program ten zarządza dozowaniem pofermentu do reaktora waloryzacji VR01 oraz sterowaniem pompą SP02, która między innymi odpowiada za transport odwodnionego pofermentu do reaktora waloryzacji VR01. Logika sterowania pompą SP02 jest realizowana za pomocą funktion block FB_DriveInv, która obsługuje komunikację Modbus RTU z falownikiem i realizuje funkcje sterowania prędkością.

### Napędy

W węźle nawozów mamy następujące napędy:

- napęd pompy SP02, który jest sterowany za pomocą funkcji FB_DriveInv, która komunikuje się z falownikiem przez Modbus RTU. Logika sterowania pompą SP02 jest realizowana w taki sposób, aby zabezpieczyć pompę przed suchobiegiem, ponieważ pompa ta nie jest wyposażona w czujniki informujące o poziomie osadu w koszu zasypowym. Dlatego zaimplementowano pracę interwałową polegającą na napełnianiu i opróżnianiu kosza zasypowego na podstawie dośwaidczalnie dobranych czasów napełniania i opróżniania 25minut napełniania (postój pompy) 2 minuty opróżnianie (dozowanie do reaktora przy częstości zadanej 40Hz). Ten cykl jest powtarzany tak długo jak stacja odwadniania jest w trybie pracy automatycznej i nie występują alarmy krytyczne. W przypadku dozowania do VR01 pompa jest sterowana na podstawie testu masy dozowanej do rektora granulacji VR01 który jest posadowiony an tensometrach o dokładności 1kg. Dozowanie jest realizowane do momentu osiągnięcia zadanej masy pofermentu w reaktorze VR01, która jest ustawiana przez operatora z SCADA. Po osiągnięciu zadanej masy pompa SP02 jest zatrzymywana i rozpoczyna się cykl dozowania wapna i perlitu.
- napędy dozowanie wapna i perlitu z silosów S01 i S02 jest realizowane za pomocą podajników SC01 i SC02 z napędami M01 i M02 ze stycznikami obsługiwane przez Drive typu FB_Drive_1c. Proces dozowania perlitu i wapna jest wspierany za pomocą elektrovibratorów E01 i E02 zainstalowanych w silosach S01 i S02 obsługiwancyh za pomocą FB_Drive_1c.
- napędy M01, M02 mieszadeł w reaktorze waloryzacji VR01, które są sterowane za pomocą funkcji FB_DriveInv. Oraz napęd M03 zamykania i otwierania klapy zrzutowej reaktora VR01, służącej do ewakluacji wytworzonego nawozu do big bagów. Napęd M03 jest sterowany za pomocą funkcji FB_DriveInv oraz jest wyposażony w enkoder inkrementalny do pozycjonowania zamknięcia i otwarcia klapy zrzutowej. Logika sterowania napędem M03 obejmuje funkcje pozycjonowania oraz zabezpieczenia przed kolizją i przeciążeniem.
- napęd wentylatora BL01, który odprowadza pary i gazy z reaktora waloryzacji VR01, jest sterowany za pomocą funkcji FB_Drive_1C. Wentylator BL01 jest załączany tylko podczas mieszania reagentów w reaktorze VR01, aby zapewnić odpowiednią wentylację i odprowadzenie gazów. Logika sterowania wentylatorem BL01 jest zsynchronizowana z logiką sterowania mieszadłami M01 i M02, tak aby wentylator był załączany tylko wtedy, gdy mieszadła są włączone i proces mieszania jest aktywny.

### Sensory

W węźle nawozów mamy następujące sensory:

- fbTempSens_H01 czujnik temperatury w hali produkcji nawozów która jest ogrzewana za pomocą ciepła odpadowego z kogeneratora CHP01.
- WeightSens_VR01 przetwornik wagi SIMEX SWI-94 do ważenia osadów odwodnionych, które trafiają do reaktora waloryzacji VR01. Dane z tego przetwornika są odczytywane przez Modbus RTU i mapowane do struktury WeightSens_VR01 w GVL_Fertilizer.
- czujniki rotacyjne poziomu wapna i perlitu w silosach S01 i S02 z wyjsciem NC, które są odczytywane jako sygnały dwustanowe w PLC i mapowane do struktur danych w GVL_Fertilizer. Do obsługi tych czujników zastosowano Function Block FB_ContactNC.
- położenie zaworów ręcznych ZR01 i ZR02, które są odczytywane jako sygnały dwustanowe w PLC i mapowane do struktur danych w GVL_Fertilizer. Do obsługi tych czujników zastosowano Function Block FB_ContactNCO. Waro dodać że zawory te są ustawiane przez operatora ręcznie z poziomu SCADA. Nie są to fizyczne wejścia do PLC.

## Pochodnia (ADS/struktury)
- ADS_BiogasTorch
- ST_TorchAdsStruct
- Pola sterowania: bStartButton, bStopButton, bResetButton
- Pola statusowe: wStatus, sStateLabel, sStatusLabel

## Kontrakty interfejsow
- PLC -> SCADA przez OPC UA (struktury w GVL)
- Modbus dla urzadzen zewnetrznych (analizator, inwertery)

## Zasady rozwoju kodu
1. Utrzymuj rozdzial warstw (sprzet, sterowanie, SCADA)
2. Nie lam kompatybilnosci istniejacych struktur GVL bez planu migracji
3. Dodawaj nowe sygnaly w grupach procesowych i dokumentuj mapowanie

## Otwarte punkty do uzupelnienia
- Lista wszystkich sygnalow krytycznych z typami i jednostkami
- Priorytety alarmow PLC (A1/A2/A3)
- Macierz testow FAT/SAT dla glownej logiki
