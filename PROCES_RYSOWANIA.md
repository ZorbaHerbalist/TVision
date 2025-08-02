# Proces Rysowania Zawartości Ekranu w Bibliotece TVision

## Spis Treści
1. [Wprowadzenie](#wprowadzenie)
2. [Architektura Systemu Rysowania](#architektura-systemu-rysowania)
3. [Hierarchia Klas i Ich Role](#hierarchia-klas-i-ich-role)
4. [Proces Rysowania - Szczegółowy Przepływ](#proces-rysowania---szczegółowy-przepływ)
5. [Kiedy Jest Wywoływany Proces Rysowania](#kiedy-jest-wywoływany-proces-rysowania)
6. [Buforowanie i Optymalizacja](#buforowanie-i-optymalizacja)
7. [Przykłady Kodu](#przykłady-kodu)
8. [Zaawansowane Mechanizmy](#zaawansowane-mechanizmy)

## Wprowadzenie

Biblioteka TVision (Turbo Vision) implementuje zaawansowany system rysowania interfejsu użytkownika w trybie tekstowym. System ten opiera się na hierarchicznej strukturze widoków (views) i wykorzystuje techniki takie jak double-buffering, clipping i zarządzanie zdarzeniami do zapewnienia płynnego i efektywnego renderowania.

## Architektura Systemu Rysowania

### Podstawowe Założenia

System rysowania w TVision opiera się na następujących zasadach:

1. **Hierarchia Widoków**: Każdy element interfejsu jest reprezentowany przez obiekt dziedziczący z klasy `TView`
2. **Zarządzanie Z-Order**: Widoki są organizowane w kolejności warstw (z-order) 
3. **Clipping**: Każdy widok rysuje tylko w swoim obszarze, nie naruszając innych widoków
4. **Event-Driven**: Rysowanie następuje w odpowiedzi na zdarzenia systemowe
5. **Buforowanie**: Wykorzystanie buforów ekranu dla optymalizacji wydajności

### Struktura Klas

```
TObject
  └── TView (podstawowa klasa dla wszystkich elementów wizualnych)
      ├── TGroup (kontener dla innych widoków)
      │   ├── TWindow (okno z ramką i tytułem)
      │   ├── TDesktop (pulpit aplikacji)
      │   └── TProgram/TApplication (główna aplikacja)
      ├── TFrame (ramka okna)
      ├── TScrollBar (pasek przewijania)
      ├── TBackground (tło)
      └── inne komponenty UI
```

## Hierarchia Klas i Ich Role

### TView - Podstawowa Klasa Widoku

**Lokalizacja**: `VIEWS.PAS` (linie 204-292)

`TView` jest fundamentalną klasą dla wszystkich elementów wizualnych. Zawiera:

**Kluczowe Pola**:
- `Origin: TPoint` - pozycja widoku względem rodzica
- `Size: TPoint` - rozmiar widoku
- `State: Word` - stan widoku (widoczny, aktywny, itp.)
- `Owner: PGroup` - wskaźnik do rodzica

**Główne Metody Rysowania**:
- `Draw()` - wirtualna metoda implementująca logikę rysowania
- `DrawView()` - metoda sterująca procesem rysowania
- `WriteBuf()` - niskopoziomowe pisanie do bufora ekranu
- `WriteChar()` - pisanie pojedynczych znaków
- `WriteStr()` - pisanie stringów

```pascal
procedure TView.Draw;
var
  B: TDrawBuffer;
begin
  MoveChar(B, ' ', GetColor(1), Size.X);
  WriteLine(0, 0, Size.X, Size.Y, B);
end;
```

### TGroup - Zarządzanie Kolekcjami Widoków

**Lokalizacja**: `VIEWS.PAS` (linie 431-483)

`TGroup` rozszerza `TView` o możliwość zawierania innych widoków:

**Kluczowe Pola**:
- `Last: PView` - wskaźnik do ostatniego widoku w liście
- `Current: PView` - aktualnie wybrany widok
- `Buffer: PVideoBuf` - bufor ekranu dla grupy

**Metody Zarządzania Rysowaniem**:
- `Redraw()` - wymusza ponowne narysowanie wszystkich widoków
- `DrawSubViews()` - rysuje wszystkie widoki podrzędne
- `Lock()/Unlock()` - blokowanie/odblokowanie rysowania

**Cykl Życia Rysowania w TGroup**:

1. **GetBuffer()** - Alokacja bufora jeśli potrzeba
2. **Lock()** - Zablokowanie operacji rysowania
3. **ForEach(@DrawOperation)** - Wykonanie operacji na wszystkich widokach podrzędnych
4. **Unlock()** - Odblokowanie i wywołanie DrawView() jeśli potrzeba

```pascal
procedure TGroup.SetState(AState: Word; Enable: Boolean);
// ...
case AState of
  sfActive, sfDragging:
    begin
      Lock;
      ForEach(@DoSetState);
      Unlock;
    end;
  sfExposed:
    begin
      ForEach(@DoExpose);
      if not Enable then FreeBuffer;
    end;
end;
```

### Proces DrawSubViews

```pascal
procedure TGroup.DrawSubViews(P, Bottom: PView);
begin
  if P <> nil then
    while P <> Bottom do
    begin
      P^.DrawView;
      P := P^.NextView;
    end;
end;
```

## Proces Rysowania - Szczegółowy Przepływ

### 1. Inicjalizacja Rysowania

Proces rozpoczyna się od wywołania `DrawView()` na widoku głównym:

```pascal
procedure TView.DrawView;
begin
  if Exposed then
  begin
    Draw;
    DrawCursor;
  end;
end;
```

### 2. Sprawdzenie Widoczności (Exposed)

Funkcja `Exposed()` (linie 1249-1354) sprawdza, czy widok jest rzeczywiście widoczny:

**Kryteria Widoczności**:
- Stan `sfExposed` jest ustawiony
- Widok ma niezerowy rozmiar
- Nie jest całkowicie zasłonięty przez inne widoki

### 3. Algorytm Clipping

System implementuje zaawansowany algorytm clipping, który:

1. **Oblicza Region Clipping**: Określa obszar, w którym widok może rysować
2. **Sprawdza Przecięcia**: Wykrywa kolizje z innymi widokami
3. **Dzieli Obszar**: Rozdziela obszar na fragmenty nie zasłonięte przez inne widoki

### 4. Rysowanie Hierarchiczne

Dla grup (`TGroup`), proces przebiega rekurencyjnie:

```pascal
procedure TGroup.Draw;
var
  R: TRect;
begin
  if Buffer = nil then
  begin
    GetBuffer;
    if Buffer <> nil then
    begin
      Inc(LockFlag);
      Redraw;
      Dec(LockFlag);
    end;
  end;
  if Buffer <> nil then WriteBuf(0, 0, Size.X, Size.Y, Buffer^) else
  begin
    GetClipRect(Clip);
    Redraw;
    GetExtent(Clip);
  end;
end;
```

**Kluczowe Metody Niskopoziomowe**:

- `WriteBuf(X, Y, W, H: Integer; var Buf)` - Zapisuje blok danych do bufora ekranu
- `WriteChar(X, Y: Integer; C: Char; Color: Byte; Count: Integer)` - Pisanie znaków
- `WriteLine(X, Y, W, H: Integer; var Buf)` - Pisanie linii z bufora
- `WriteStr(X, Y: Integer; Str: String; Color: Byte)` - Pisanie stringa

**Proces WriteView** (implementowany w assemblerze):
1. Sprawdzenie granic widoku
2. Obliczenie współrzędnych globalnych
3. Zastosowanie clipping
4. Rekurencyjne przechodzenie przez widoki zasłaniające
5. Właściwe pisanie do bufora ekranu z obsługą snow effect

### 5. Finalizacja i Buforowanie

Na końcu procesu:
- Zawartość jest zapisywana do bufora ekranu
- Kursor jest aktualizowany jeśli potrzeba
- Stan `Exposed` jest odpowiednio ustawiony

## Kiedy Jest Wywoływany Proces Rysowania

### 1. Automatyczne Wyzwalacze

**Zdarzenia Systemowe**:
- Zmiana rozmiaru okna: `ChangeBounds()`
- Pokazanie widoku: `Show()`
- Ukrycie widoku: `Hide()`
- Zmiana stanu: `SetState()`

**Zdarzenia Użytkownika**:
- Przemieszczenie okna
- Przełączenie między oknami
- Zmiana rozmiaru przez użytkownika

### 2. Programowe Wywołania

```pascal
// Bezpośrednie wywołanie rysowania
View^.DrawView;

// Wymuszenie pełnego przerysowania
Group^.Redraw;

// Przerysowanie po zmianie zawartości
Invalidate; // (w niektórych implementacjach)
```

### 3. Cykl Zdarzeń Aplikacji

W głównej pętli aplikacji (`TProgram.Execute`):

```pascal
function TGroup.Execute: Word;
var
  E: TEvent;
begin
  repeat
    EndState := 0;
    repeat
      GetEvent(E);
      HandleEvent(E);
      if E.What <> evNothing then EventError(E);
    until EndState <> 0;
  until Valid(EndState);
  Execute := EndState;
end;
```

**Typy Zdarzeń Wywołujących Rysowanie**:

- `evMouseDown`, `evMouseMove` - Zdarzenia myszy
- `evKeyDown` - Zdarzenia klawiatury  
- `evCommand` - Komendy systemowe (cmResize, cmNext, itp.)
- `evBroadcast` - Komunikaty broadcast między widokami

**Metoda SetState i jej wpływ na rysowanie**:

```pascal
procedure TView.SetState(AState: Word; Enable: Boolean);
begin
  // ... 
  if AState and sfVisible <> 0 then
  begin
    if Enable then DrawShow(LastView)
    else DrawHide(LastView);
  end;
  // ...
end;
```

## Buforowanie i Optymalizacja

### 1. Bufor Ekranu (Screen Buffer)

**Lokalizacja**: `DRIVERS.PAS` (linie 175-181)

```pascal
ScreenBuffer: Pointer;  // Wskaźnik do bufora ekranu
CheckSnow: Boolean;     // Kontrola snow effect dla starych kart CGA
```

### 2. Bufor Grupy (Group Buffer)

Grupy mogą mieć własne bufory dla optymalizacji:

```pascal
procedure TGroup.GetBuffer; assembler;
asm
    LES DI,Self
    TEST    ES:[DI].State,sfExposed
    JZ  @@1
    TEST    ES:[DI].Options,ofBuffered
    JZ  @@1
    // ... kod alokacji bufora
@@1:
end;
```

**Warunki Alokacji Bufora**:
- Grupa jest widoczna (`sfExposed`)
- Ma włączoną opcję buforowania (`ofBuffered`)
- Rozmiar nie przekracza 32768 bajtów

### 3. Mechanizm Lock/Unlock

Zapobiega wielokrotnemu rysowaniu podczas operacji hurtowych:

```pascal
procedure TGroup.Lock;
begin
  if (Buffer <> nil) or (LockFlag <> 0) then Inc(LockFlag);
end;

procedure TGroup.Unlock;
begin
  if LockFlag <> 0 then
  begin
    Dec(LockFlag);
    if LockFlag = 0 then DrawView;
  end;
end;
```

## Przykłady Kodu

### Przykład 1: Prosty Widok z Własnym Rysowaniem

```pascal
type
  PMyView = ^TMyView;
  TMyView = object(TView)
    procedure Draw; virtual;
  end;

procedure TMyView.Draw;
var
  B: TDrawBuffer;
  Color: Byte;
begin
  Color := GetColor(1);
  
  // Wypełnij tło
  MoveChar(B, ' ', Color, Size.X);
  WriteLine(0, 0, Size.X, Size.Y, B);
  
  // Narysuj ramkę
  MoveChar(B, #196, Color, Size.X);  // Górna linia
  WriteLine(0, 0, Size.X, 1, B);
  WriteLine(0, Size.Y-1, Size.X, 1, B);  // Dolna linia
  
  // Boże strony
  WriteChar(0, 1, #179, Color, Size.Y-2);    // Lewa
  WriteChar(Size.X-1, 1, #179, Color, Size.Y-2);  // Prawa
  
  // Narysuj tekst w środku
  WriteStr(1, 1, 'Mój Widok', Color);
end;
```

### Przykład 2: Wykorzystanie TDrawBuffer

```pascal
procedure DrawStatusLine;
var
  B: TDrawBuffer;
  I: Integer;
  Color: Byte;
begin
  Color := GetColor(1);
  
  // Przygotuj bufor
  MoveChar(B, ' ', Color, Size.X);
  
  // Dodaj tekst
  MoveStr(B[1], 'F1-Pomoc', Color);
  MoveStr(B[20], 'F10-Menu', Color);
  MoveStr(B[40], 'Alt+X-Wyjście', Color);
  
  // Wyślij do ekranu
  WriteLine(0, 0, Size.X, 1, B);
end;
```

### Przykład 3: Obsługa Zdarzenia Resize

```pascal
procedure TMyWindow.HandleEvent(var Event: TEvent);
begin
  inherited HandleEvent(Event);
  
  if Event.What = evCommand then
    case Event.Command of
      cmResize:
        begin
          // Okno zostało przeskalowane
          DrawView;  // Przerysuj zawartość
          ClearEvent(Event);
        end;
    end;
end;
```

## Zaawansowane Mechanizmy

### 1. Zarządzanie Z-Order

Widoki w grupie są organizowane w listę cykliczną:

```pascal
// Przeniesienie na front
procedure TView.MakeFirst;
begin
  PutInFrontOf(Owner^.First);
end;

// Przeniesienie przed określony widok
procedure TView.PutInFrontOf(Target: PView);
// ... kompleksowa logika zarządzania kolejnością
```

### 2. Obsługa Cieni (Shadows)

Okna mogą mieć cienie dla efektu 3D:

```pascal
// W SetState:
if State and sfShadow <> 0 then DrawUnderView(True, LastView);

// Rozmiar cienia
ShadowSize: TPoint = (X: 2; Y: 1);
ShadowAttr: Byte = $08;
```

### 3. Kolorowanie i Palety

System kolorów oparty na paletach:

```pascal
function TView.GetColor(Color: Word): Word;
// Mapowanie koloru logicznego na fizyczny
// Uwzględnia paletę widoku i jego rodziców
```

### 4. Obsługa Snow Effect

Dla starych kart CGA system zawiera specjalną obsługę:

```pascal
// Fragment WriteView (assembler)
OR AL,AL
JNE @@60
// ... normalne kopiowanie
@@60:
// ... obsługa snow effect z synchronizacją vertical retrace
```

## Podsumowanie

System rysowania TVision jest zaawansowanym mechanizmem, który:

1. **Hierarchicznie organizuje** elementy interfejsu
2. **Optymalizuje wydajność** poprzez buforowanie i clipping
3. **Zapewnia spójność** poprzez automatyczne przerysowywanie
4. **Obsługuje różne tryby** video i karty graficzne
5. **Implementuje zaawansowane efekty** jak cienie i różne palety kolorów

Kluczem do zrozumienia systemu jest świadomość, że każdy element interfejsu jest obiektem z własną logiką rysowania, a system automatycznie zarządza złożonymi aspektami jak kolizje, clipping i optymalizacja wydajności.

## Sekwencja Czasowa Rysowania

### Typowy Przepływ dla Zdarzenia Użytkownika:

1. **Odbior Zdarzenia** - `GetEvent()` w głównej pętli
2. **Trasowanie Zdarzenia** - `HandleEvent()` znajduje odpowiedni widok
3. **Przetwarzanie** - Widok modyfikuje swój stan
4. **Wywołanie SetState** - Jeśli zmienia się widoczność
5. **Sprawdzenie Lock** - Czy grupa jest zablokowana
6. **DrawView** - Jeśli nie zablokowana, natychmiastowe rysowanie
7. **Propagacja** - Powiadomienie widoków podrzędnych o zmianie

### Kiedy NIE następuje rysowanie:

- Widok jest ukryty (`sfVisible` = false)
- Grupa jest zablokowana (`LockFlag` > 0)
- Widok nie jest odsłonięty (`Exposed()` = false)  
- Aplikacja jest zminimalizowana lub nieaktywna

Tego typu architektura zapewnia, że interface użytkownika jest zawsze aktualny i responsywny, przy jednoczesnej optymalizacji wydajności poprzez inteligentne zarządzanie procesami rysowania.