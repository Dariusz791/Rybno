# 03 - Wizualizacja SCADA (Promotic)

## Cel dokumentu
Ten dokument opisuje strukture obiektow SCADA, nazewnictwo i mapowanie na dane PLC.

## Glowna konwencja obiektowa
- Kotly/{NazwaKotla}{SymbolKotla}
- Pochodnie/{NazwaPochodni}{SymbolPochodni}
- Napedy/{NazwaNapedu}{SymbolNapedu}

## Panele i stacyjki
- StacyjkaWezlaCiepla.xml
- StacyjkaPochodni.xml
- MotorStd.xml

## Przyklady sciezek zmiennych
### Wezel ciepla / reaktor
- ../Kotly/{NazwaKotla}{SymbolKotla}/Data/#vars/bReaktorR01
- ../Kotly/{NazwaKotla}{SymbolKotla}/Data/#vars/rSpR01
- ../Kotly/{NazwaKotla}{SymbolKotla}/Data/#vars/rCvR01

### Pochodnia
- ../Pochodnie/{NazwaPochodni}{SymbolPochodni}/Data/#vars/CisnienieBiogazu
- ../Pochodnie/{NazwaPochodni}{SymbolPochodni}/Data/#vars/PoziomOtwarciaZaworu
- ../Pochodnie/{NazwaPochodni}{SymbolPochodni}/Data/#vars/sStatus
- ../Pochodnie/{NazwaPochodni}{SymbolPochodni}/Data/#vars/wStatus
- ../Pochodnie/{NazwaPochodni}{SymbolPochodni}/Data/#vars/bAutoManualSwitch
- ../Pochodnie/{NazwaPochodni}{SymbolPochodni}/Data/#vars/bStartButton
- ../Pochodnie/{NazwaPochodni}{SymbolPochodni}/Data/#vars/bStopButton
- ../Pochodnie/{NazwaPochodni}{SymbolPochodni}/Data/#vars/bResetButton

## Standard prezentacji alarmow i stanow
- Kolor czerwony: alarm aktywny
- Kolor zolty: ostrzezenie / stan przejsciowy
- Kolor zielony: praca poprawna
- Kolor szary: brak komunikacji / nieaktywny

## Dobre praktyki
1. Uzywaj tych samych nazw logicznych w PLC i SCADA tam, gdzie to mozliwe
2. Nie duplikuj logiki procesowej po stronie panelu
3. Trzymaj status i komendy w czytelnych grupach Data/#vars

## Otwarte punkty do uzupelnienia
- Slownik tekstow alarmowych (PL/EN)
- Macierz uprawnien operator/serwis/admin dla komend
- Lista ekranow krytycznych do testow odbiorowych
