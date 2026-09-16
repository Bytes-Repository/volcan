# volcan

**`volcan`** to biblioteka dla **H#**, będąca odpowiednikiem rustowego
[smithay](https://github.com/Smithay/smithay) — zestaw klocków do pisania
własnych kompozytorów Wayland: Wayland core, XWayland, DRM/KMS, GBM/EGL,
libinput. Cała logika jest napisana **w 100% w H#** — nie ma tu ani
linijki Rusta czy C. `extern static [c, "..."]` w `src/ffi/*.h#` wiąże
się bezpośrednio z gotowymi bibliotekami systemowymi (`libwayland-server`
jako dokumentacja ABI, `libdrm`, `libgbm`, `libEGL`, `libinput`, `libxcb`,
`libudev`, `libxkbcommon`) — dokładnie tymi, na których smithay/wlroots
budują swój stack pod maską.

## Status: zweryfikowane prawdziwym parserem H#

Wszystkie 44 pliki tego pakietu **parsują się bezbłędnie** pod realnym
kompilatorem H# (`hsharp-parser`, zbudowanym lokalnie z oficjalnego
źródła i uruchomionym jako niezależny walidator składni — nie jest to
odgadywanie na podstawie przykładów). W trakcie tej weryfikacji
odkryto i naprawiono kilka nieoczywistych, fundamentalnych zasad H#,
których nie da się wywnioskować z samej dokumentacji/przykładów:

- **Lokalne pliki projektu importuje się przez bezimienne `mod nazwa`**,
  nie przez `use "..."` (ta forma jest zarezerwowana dla `std -> `,
  `bytes -> `, `python -> `, `github -> `). `mod nazwa` szuka
  `nazwa.h#`/`nazwa/mod.h#`/`nazwa/main.h#`, najpierw w katalogu pliku,
  który go deklaruje. Stąd struktura `ffi/mod.h#`, `protocols/mod.h#`,
  `backend/mod.h#`, `renderer/mod.h#` — agregatory per katalog, spięte
  w `src/volcan.h#`.
- **Funkcje `extern` i stałe `pub const` NIE są nigdy prefiksowane**
  przy imporcie modułu — wołane są zawsze bez aliasu (`drmOpen(...)`,
  `AF_UNIX`), podczas gdy zwykłe `fn`/`struct`/`enum` SĄ prefiksowane
  (`wire::peek_u32(...)`). To rozróżnienie widać w każdym pliku.
- **H# nie ma mutowalnych zmiennych na poziomie modułu** — tylko
  `const`/immutable `let` poza funkcjami. Stan trzeba nosić jawnie
  przez struktury (patrz `cs::VolcanState.next_serial` zamiast
  globalnego licznika).
- Sygnatury funkcji w blokach `extern static [...] is ... end` muszą
  mieścić się w jednej linii (parser nie pomija nowej linii po
  przecinku wewnątrz listy parametrów extern).
- Komentarze istnieją tylko jako `;; ...` — brak `/* ... */`.
- `write` jest zarezerwowanym słowem kluczowym.
- Stałe (`pub const`) są globalne bez namespacingu — dwie stałe o tej
  samej nazwie w różnych plikach to realna kolizja (znaleziono i
  naprawiono jedną: `ANCHOR_TOP`/`BOTTOM`/`LEFT`/`RIGHT` w `wm.h#`
  kolidowały z tymi samymi nazwami w `protocols/layer_shell.h#` —
  przemianowane na `POS_ANCHOR_*`).

**Czego NIE zweryfikowano** (poza zasięgiem tego środowiska): pełny
`bytes build` wymaga kompilatora H# z LLVM 21, którego nie dało się tu
zainstalować (potrzebuje `apt.llvm.org`, spoza dozwolonych domen sieci
w tym środowisku) — więc mangling całego programu, typecheck i
faktyczna budowa binarki pozostają nieprzetestowane end-to-end.
Składnia każdego pojedynczego pliku jest jednak zweryfikowana
narzędziowo, nie na wiarę.

## Dlaczego bez plików `.xml`?

Standardowo protokoły Wayland (`wayland.xml`, `xdg-shell.xml`, ...) są
kompilowane przez `wayland-scanner` do C-owych tablic `wl_interface`/
`wl_message`. `volcan` pomija ten krok: `protocols/wire.h#` układa
DOKŁADNIE te same bajty w pamięci, ale na podstawie zwykłych danych H#
(`protocols/core_wl.h#`, `xdg.h#`, `layer_shell.h#`, `extras.h#`).

## Dlaczego pętla `poll()`, a nie callbacki libwayland?

**Kluczowe odkrycie architektoniczne tego projektu:** H# nie ma sposobu
na wyeksportowanie funkcji H# jako wskaźnika na funkcję C (potwierdzone
czytaniem `source-code/parser` — jedyne "callbacki" w bibliotece
standardowej, np. `std/websocket.h#`, to zwykłe stringi, nie adresy).
To wyklucza cały model "C wywołuje H#": `wl_global_create`'s bind
callback, `wl_event_loop_add_fd`'s callback, `libinput`'s
`open_restricted`/`close_restricted`, `libseat`'s listener — żaden z
nich nie da się zarejestrować z czystego H#.

Rozwiązanie: `volcan` NIGDY nie polega na tym, że C wywołuje H#.
Zamiast tego cała biblioteka to jedna pętla `poll()` (`src/volcan.h#`)
odpytująca wszystkie deskryptory naraz (gniazdo nasłuchujące, każdy
klient, DRM, libinput, udev, XWayland), a `src/dispatcher.h#` sam
parsuje protokół Wayland-wire bezpośrednio z gniazda klienta
(`protocols/codec.h#`) — bez oddawania tego `libwayland-server`'s
dispatcherowi. `protocols/registry.h#`/`ffi/ws.h#` (natywna ścieżka
przez `wl_global_create`) zostają w drzewie jako udokumentowana
alternatywa na wypadek, gdyby kompilator H# kiedyś dostał eksport
wskaźników funkcji.

## Architektura

```
volcan.hk                      manifest pakietu (bytes.io)
src/
  volcan.h#                    ENTRY: mod-y spinajace caly projekt, API publiczne, petla poll()
  ffi/                         surowe bindingi extern static [c, ...] (+ mod.h# agregator)
    ws.h#, drm.h#, gbm.h#, egl.h#, li.h#, xcb.h#, libc.h#, sock.h#, udev.h#, xkb.h#, libseat.h#
  protocols/                   (+ mod.h# agregator)
    wire.h#      silnik: dane H# -> wl_interface w pamieci (sciezka natywna/alt)
    core_wl.h#   rdzen Wayland (compositor/surface/shm/seat/output/...)
    xdg.h#       okna (toplevel/popup) + dekoracje
    layer_shell.h# panele/paski/tla (waybar, swaybg, ...)
    extras.h#    linux-dmabuf, presentation-time, viewporter
    args.h#      typ WireArg + kodowanie wl_argument (sciezka natywna/alt)
    codec.h#     WLASNY kodek wire-protokolu (zywa sciezka I/O)
    lookup.h#    sygnatury requestow po (interfejs, opcode)
    registry.h#  wl_global_create (SCIEZKA NATYWNA/ALT, nieuzywana przez petle glowna)
  backend/                      (+ mod.h# agregator)
    drmb.h#      skan GPU (peek na drmModeRes/Connector), atomic KMS, multi-GPU
    inputb.h#    dispatch libinput -> seat.h#
    udevmon.h#   hot-plug bez callbackow (poll-based)
    session.h#   model sesji VT-switch (czesciowo zablokowany, patrz komentarz w pliku)
    keymap.h#    XKB: budowa mapy klawiatury, modyfikatory
  renderer/gles.h#              (+ mod.h# agregator) kompozycja: teksturowane quady, dmabuf import
  display.h#     reczne gniazdo UNIX $WAYLAND_DISPLAY
  cs.h#          stan globalny: klienci, powierzchnie, fokus, serial
  client.h#      stan polaczenia: fd, tabela obiektow, bufor odbioru
  dispatcher.h#  parsuje wire-protokol z gniazda, wykonuje logike
  surface.h#     attach/damage/commit
  output.h#      model monitora + auto-layout wielu wyjsc
  seat.h#        fokus i wysylka zdarzen do klienta
  wm.h#          geometria popupow (xdg_positioner), clamp rozmiaru, configure/ack
  layer_shell_logic.h#  strefy wykluczenia, zakotwiczenie do wyjscia
  xwl.h#         spawn Xwayland (fork/socketpair), rozpoznawanie zdarzen XCB
examples/minimal_compositor.h#
tests/           logika bez sprzetu: wm.h#, codec.h# (uruchom: `bytes test`)
```

## Szybki start

```h#
use "bytes -> volcan/0.1.0" from "volcan"

fn main() is
    let comp = volcan::compositor("/dev/dri/card0", "pl", false, true)
    volcan::run(comp)
end
```

## Co jest tu naprawdę zaimplementowane, a co jest punktem rozbudowy

`smithay` to efekt kilku lat pracy wielu osób — `volcan` w obecnym
kształcie to spójny, zweryfikowany składniowo szkielet architektoniczny
z realnie działającą logiką w kluczowych miejscach, nie gotowy do
produkcji zamiennik po jednym `bytes build`.

**W pełni rozpisane:** cały FFI, kodek wire-protokołu z round-trip
testami, dyspozytor requestów (wl_display/registry/compositor/surface/
shm/seat/xdg_shell — podstawowy zestaw), model stanu, geometria
popupów i state machine configure/ack, strefy wykluczenia layer-shell,
skan wyjść DRM przez odczyt pamięci, pętla `poll()` bez callbacków.

**Jawnie oznaczone jako punkt rozbudowy** (wymaga dopasowania do
konkretnego sprzętu/testów na żywym kompilatorze): pełne property IDs
przy atomowym commit KMS, parsowanie `drmModeModeInfo[]` (tryby
wyjścia), `libinput`/`libseat` bez roota (blokowane brakiem eksportu
wskaźników funkcji w H# — udokumentowany plan z D-Bus/logind jako
obejście), ICCCM/EWMH przez XWayland, clipboard bridge X11↔Wayland.
