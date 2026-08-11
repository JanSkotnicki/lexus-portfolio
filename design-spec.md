# Design Spec

## 1. Typografia nagłówków

- Font: Archivo (Google Fonts), waga 600
- letter-spacing: -0.02em
- line-height: 1.12
- Zastępuje obecny krój w nagłówkach. Tekst ciągły zostaje bez zmian.

## 2. Paleta

- tło: `#F4F2ED`
- tekst główny: `#1A1917`
- tekst drugorzędny: `#6B6862`
- akcent: `#8C7A5B`
- Zastępuje obecne czyste biel/czerń.

## 3. Layout podstron case study

- tytuł: duży, pełna szerokość kolumny, wyśrodkowany, dużo powietrza wokół
- pod tytułem wiersz metadanych w jednej linii: Client / Venue / Role
- pod metadanymi akapit wprowadzający, wyśrodkowany, węższa kolumna niż tytuł
- zdjęcia wchodzą POD blokiem tekstu jako dowód, nie obok niego
- tekst prowadzi, zdjęcie potwierdza

## Bez zmian

- kolejność i rozkład sekcji na stronie głównej
- układ kafelków w sekcji Proof
- treść tekstowa wszystkich stron

## System tła

Jedno przejście na całej stronie: `#F4F2ED` od Hero przez POV, Proof i Thesis,
potem zejście do ciemnego (`#0a0a0a`) w Contact. Dawne przejścia przy sekcji
POV (jasne→czarne, czarne→jasne przy Proof) zostały usunięte w Fazie 1.

`makeBgColorSystem` pozostaje jedynym miejscem, które zapisuje
`body.style.backgroundColor` — nowe przejście dokłada stop do istniejącej
tablicy, nie tworzy drugiego ScrollTriggera (szczegóły: architecture-v2.md).
