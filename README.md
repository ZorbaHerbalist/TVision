# TVision Library

Biblioteka TVision to zaawansowany framework do tworzenia interfejsów użytkownika w trybie tekstowym dla języka Pascal (Turbo Pascal/Free Pascal).

## Dokumentacja

### [Proces Rysowania Zawartości Ekranu](PROCES_RYSOWANIA.md)

Szczegółowy dokument opisujący:
- Jak przebiega proces rysowania w aplikacjach TVision
- Kiedy jest wywoływany proces rysowania
- Hierarchię klas i ich role w systemie renderowania
- Przykłady kodu i zaawansowane mechanizmy
- Optymalizację wydajności i buforowanie

## Struktura Projektu

- `src/` - Kod źródłowy biblioteki TVision
  - `VIEWS.PAS` - Podstawowe klasy widoków i zarządzanie rysowaniem
  - `DRIVERS.PAS` - Niskopoziomowe sterowniki ekranu i zdarzeń
  - `APP.PAS` - Framework aplikacji i zarządzanie pulpitem
  - `MENUS.PAS` - System menu
  - `DIALOGS.PAS` - Okna dialogowe
  - `EDITORS.PAS` - Edytory tekstu
  - inne pliki modułów

## Wykorzystanie

Biblioteka TVision pozwala na tworzenie zaawansowanych aplikacji konsolowych z:
- Oknami z możliwością przesuwania i zmiany rozmiaru
- Systemem menu
- Oknami dialogowymi
- Edytorami tekstu
- Paskami przewijania
- Obsługą myszy i klawiatury

## Historia

Oryginalnie stworzona przez Borland dla Turbo Pascal 7.0 w 1992 roku.