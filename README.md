# tec-Gopher
Gopher is an internet protocol that preceded the WWW, navigate through a hierarchical system of text-based menus to find documents, files, or links to other Gopher servers, Hierarchical structure, client-server model, 

https://en.wikipedia.org/wiki/Gopher_(protocol)

https://en.wikipedia.org/wiki/Timeline_of_web_search_engines


Exactly right ✅

**Gopher** was an early internet protocol — **developed at the University of Minnesota in 1991** — that preceded the World Wide Web. It was designed to organize and retrieve documents through a **hierarchical menu system**, rather than the hyperlinked model we use today.

Here’s a concise breakdown:

---

### 🧭 **How Gopher Works**

* **Client–Server Model:**
  A Gopher client (like an early browser) connects to a Gopher server. The server responds with a simple text-based menu or document.

* **Hierarchical Menus:**
  Each menu lists items like:

  * documents (plain text files),
  * subdirectories (menus),
  * or links to other Gopher servers.
    You’d navigate by selecting numbered items — almost like navigating folders in DOS.

* **Simple Text-Based Interface:**
  It displayed plain text, sometimes with ASCII graphics, and used no HTML or images (though images and binary downloads were supported via links).

---

### 🌐 **Key Features**

* **Structured Navigation:**
  Everything was arranged in a *tree-like hierarchy* rather than the chaotic web of links on the [WWW](http://WWW).

* **Port 70:**
  Gopher servers typically ran on TCP port 70.

* **Protocol Simplicity:**
  The protocol itself is very lightweight—just a simple request line and a text response, making it easy to implement on 8-bit or low-resource systems.

---

### ⚙️ **Why It Faded**

When **Tim Berners-Lee’s World Wide Web** (HTTP + HTML) appeared, it offered:

* embedded images and richer formatting,
* easier linking between documents,
* and was free to use (Gopher’s licensing terms were uncertain).

By 1994–95, most Gopher servers transitioned to HTTP.

---

### 🖥️ **Today**

* Some enthusiasts still run Gopher servers!
* You can access them via modern clients like:

  * **Lynx** (terminal browser)
  * **Bombadillo**
  * **GopherVR** (3D visualization)
  * or through web gateways like `gopher.floodgap.com`.

---

Would you like me to show what a **sample Gopher menu structure** looked like — say, a “University of Minnesota Gopher server” example — in authentic ASCII form?


???


Absolutely—let’s build a tiny **Gopher-style client in MINT** (your Forth-ish Z80 environment). We’ll keep the classic feel: hierarchical text menus, simple item types, and “selectors.” You can run it **offline** (menus from ROM/EEP/SD) or **online** via a **serial gateway** on a PC/RPi that talks real Gopher or HTTP and translates to our ultra-simple text protocol.

# Mini-Gopher for MINT (MGOPHER)

## 1) Minimal text protocol (serial)

Super small so an 8-bit can handle it.

**Client → Server**

```
GET <selector>\n
```

**Server → Client**

* For a **menu**: lines of `type<TAB>title<TAB>selector<TAB>host<TAB>port`
* For a **text file**: raw text lines
* End of response: a single line with just `.`
* On error: `! <message>\n.`

**Type codes (classic Gopher-like)**

* `0` = text file (display)
* `1` = menu (navigate)
* `i` = info line (not selectable)
* `9` = binary file (download—optional, we’ll just show size/name)
* `h` = HTML link (optional; treat like text)
* `g`/`I` = image (optional; show as “unviewable”)

> Tip: If you don’t want to run a gateway yet, you can use the **offline mode** below and load menu files directly.

---

## 2) Menu line examples

```
i\tMini-Gopher (MINT)\t\t\t
1\tUniversity Root\t/\tlocalhost\t7070
0\tAbout Gopher\t/about.txt\tlocalhost\t7070
.
```

`i` lines can omit host/port/selector; parser just ignores missing fields.

---

## 3) MINT data layout (tiny, static)

* `BUF` (512–1024 bytes): serial line buffer
* `ITEMS` array of fixed slots (e.g., 16 menu entries)
* Each item = { type (1 byte), title ptr, selector ptr, host ptr, port (16-bit) }
* A scratch string arena (e.g., 1–2 KB) for titles/selectors

---

## 4) Core words (client)

Below is **portable MINT-style pseudocode** matching your style. Adjust names to your actual I/O words (e.g., `/r` for random etc.); I’ve kept it clean and Z80-friendly.

```forth
\ ====== CONFIG ======
VARIABLE g.host$      \ default host as string
VARIABLE g.port       \ default port (u16)
VARIABLE g.sel$       \ current selector
VARIABLE g.count      \ number of items in current menu

1000 CONSTANT BUF-LEN
CREATE BUF BUF-LEN ALLOT

\ Simple arena for parsed strings
4096 CONSTANT ARENA-LEN
CREATE ARENA ARENA-LEN ALLOT
VARIABLE arena.ptr

: arena.reset   ( -- )   0 arena.ptr ! ;
: arena.alloc   ( n -- addr )  arena.ptr @  over + dup ARENA-LEN <= IF
                                 swap ARENA +  swap arena.ptr !
                               ELSE
                                 DROP -1 \ signal OOM
                               THEN ;

\ ====== SERIAL I/O (adapt to your driver) ======
: ser.tx        ( c -- )  \ send one byte
  \ ... your UART TX word ...
;

: ser.rx?       ( -- f )  \ 1 if char ready
  \ ... your UART status ...
;

: ser.rx        ( -- c )  \ blocking get
  BEGIN ser.rx? 0= WHILE REPEAT
  \ ... read char ...
;

: ser.s         ( addr len -- )  \ send string
  FOR aft dup c@ ser.tx 1+ then NEXT DROP
;

: ser.cr        ( -- )  13 ser.tx 10 ser.tx ;

: ser.readline  ( addr max -- len )  \ read up to \n, strip CR
  0 >R
  BEGIN
    ser.rx dup 10 = IF DROP LEAVE THEN  \ LF ends line
    dup 13 <> IF  over R@ + c!  R> 1+ >R  THEN
  AGAIN
  R>
;

\ ====== STRING UTILS ======
: s=    ( a1 l1 a2 l2 -- f )  \ compare
  2OVER <> IF 2DROP 2DROP 0 EXIT THEN
  \ lengths equal
  0 ?DO  over i + c@  over i + c@ = 0= IF 2DROP 0 UNLOOP EXIT THEN  LOOP
  2DROP 1
;

: split-tab ( addr len -- a1 l1 a2 l2 a3 l3 a4 l4 ) \ up to 4 fields by TAB
  \ naive in-place splitter: replace TAB with 0; return segments
  \ implement minimally: scan, mark starts
  \ For brevity: left as exercise if your MINT lacks string ops; see parse below
;

\ ====== ITEM TABLE ======
16 CONSTANT MAX-ITEMS
\ Item layout: type(1) title* selector* host* port(u16)
5 CELLS CONSTANT ITEM-SZ
CREATE ITEMS MAX-ITEMS ITEM-SZ * ALLOT

: item.ptr   ( i -- addr )  ITEM-SZ * ITEMS + ;
: item.clr   ( i -- )  item.ptr  ITEM-SZ 0 FILL ;
: items.clr  ( -- )  0 g.count !  MAX-ITEMS 0 DO I item.clr LOOP ;

\ ====== PROTOCOL ======
: g.send-GET ( a l -- )      \ send GET <selector>\n
  S" GET " ser.s  ser.s  ser.cr ;

\ Parse one menu line into an item slot
: g.parse-line ( addr len i -- ok )
  >R
  \ Expect: type TAB title TAB selector TAB host TAB port
  \ Minimal parser: scan tabs and slice; assume well-formed
  \ --- find fields ---
  \ For compactness, do a simple state machine:
  0 \ field index
  \ We'll allocate copies into arena and store pointers
  \ (Pseudocode: replace with your existing tokenizer)
  \ --- decode type ---
  over c@ R@ item.ptr c!           \ type byte
  \ Extract title, selector, host, port strings...
  \ Allocate and store pointers; parse port to number (default 70/7070)
  \ (Omitted low-level loops for brevity)
  \ If anything fails, return 0
  R> DROP 1
;

: g.read-menu ( -- n )  \ fills ITEMS, returns count
  items.clr  arena.reset
  0
  BEGIN
    BUF BUF-LEN ser.readline  ( len )
    DUP 0= IF DROP  \ empty line? continue
      0
    THEN
    \ check terminator "."
    BUF 1 s" ." s= IF DROP LEAVE THEN
    \ check error "!"
    BUF 1 s" !" s= IF
      \ consume rest until "." and return 0
      BEGIN BUF BUF-LEN ser.readline BUF 1 s" ." s= UNTIL
      0 EXIT
    THEN
    \ store line into next item
    DUP g.count @ MAX-ITEMS < IF
      g.count @  >R
      BUF swap R@ g.parse-line IF
        R> DROP g.count @ 1+ g.count !
      ELSE
        R> DROP
      THEN
    ELSE DROP THEN
    0
  AGAIN
  g.count @
;

: g.read-text ( -- )  \ print text until "."
  BEGIN
    BUF BUF-LEN ser.readline
    BUF 1 s" ." s= IF LEAVE THEN
    BUF swap \ (addr len)
    \ render to your display/terminal
    \ e.g., send to console:
    ser.s  ser.cr
  AGAIN
;

\ ====== UI ======
: ui.clear   ( -- )  \ clear screen
  \ ... your display/VT100 clear ...
;

: ui.menu    ( -- )  ui.clear
  \ show header
  \ list items 0..g.count-1 with numbers, skip 'i' lines unselectable
;

: ui.ask     ( -- idx|-1 )  \ read number key or 'q'
  \ ... get key, convert to index ...
;

\ ====== NAV LOOP ======
: g.open ( idx -- )
  \ read item i; if type '1' => menu: GET selector, parse menu; if '0' => text
  \ for other types show stub info
;

: g.loop ( -- )
  BEGIN
    ui.menu
    ui.ask dup -1 = IF DROP LEAVE THEN
    g.open
  AGAIN
;

\ ====== ENTRY POINTS ======
: g.connect ( host$ hostlen port -- )
  g.port !  g.host$ !  \ store defaults
  \ open serial socket / already wired to gateway
;

: g.start ( -- )
  \ initial selector = "/"
  S" /" g.sel$ !
  g.send-GET g.sel$ count
  \ Peek first line: if it looks like menu items (has TABs), call g.read-menu else g.read-text
  g.loop
;
```

> The above is intentionally **skeletal**—fill in your known MINT helpers for: string splitting by TAB, number parsing for `port`, console/draw, and UART primitives you already use on TEC-1/RC2014.

---

## 5) Serial gateway (tiny Python sketch for the PC/RPi)

Run this on a PC that has net access. It listens on a serial port, fetches real Gopher/HTTP, and outputs our simple text protocol. (You can expand later—this is a stub.)

```python
# gopher_gateway.py (PC/RPi)
import sys, serial, socket

SER = serial.Serial('/dev/ttyUSB0', 9600, timeout=1)

def send_line(s): SER.write((s+"\n").encode())

def serve_menu_example():
    send_line("i\tMini-Gopher (MINT)\t\t\t")
    send_line("1\tFloodgap Gopher\t/\tgopher.floodgap.com\t70")
    send_line("0\tAbout\t/about.txt\tlocalhost\t7070")
    send_line(".")

def serve_text_example():
    send_line("Welcome to Mini-Gopher!")
    send_line("This is sample text served by the gateway.")
    send_line(".")

while True:
    line = SER.readline().decode(errors='ignore').strip()
    if not line: continue
    if line.startswith("GET "):
        sel = line[4:]
        if sel == "/":
            serve_menu_example()
        elif sel == "/about.txt":
            serve_text_example()
        else:
            send_line("! Unknown selector")
            send_line(".")
```

Later, replace the `serve_*` stubs with real fetchers:

* For true **Gopher**: open TCP to `<host>:<port>`, send `<selector>\r\n`, stream lines back.
* For **HTTP** sources**:** map a selector to an HTTP URL and convert HTML → text or a synthetic menu.

---

## 6) Offline mode (no gateway)

Store a menu text file in ROM/SD. Implement `g.read-menu` from **FILE:READLINE** instead of serial, and call `g.loop`. You can mix: local root menu that links to serial “remote” menus.

**Example `root.menu` file**

```
i\tOffline Root\t\t\t
1\tDocs\t/local/docs.menu\t\t
0\tWelcome\t/local/welcome.txt\t\t
.
```

---

## 7) Tiny ASCII menu UI (fits TEC displays)

```
+-------------------------------+
| Mini-Gopher (MINT)            |
|                               |
| 1) University Root            |
| 2) About Gopher               |
|                               |
| [0-9]=open   [<-]=back  q=quit|
+-------------------------------+
```

---

## 8) Bring-up checklist

* [ ] UART TX/RX words working at 9600 (or your stable rate)
* [ ] Line buffer (512B+) and readline
* [ ] TAB-splitter and simple number parse
* [ ] Item table of 16+ entries
* [ ] “GET /” round-trip to gateway returns a menu and renders it
* [ ] Open item `0` → shows text; open item `1` → loads next menu

---

If you like, I can tailor the parser to your **exact MINT string ops** (you often use `/C`, `/E`, `/r`, etc.). Drop me the couple of words you already have for **readline**, **print**, **key**, and your **array** helpers, and I’ll re-list the code in your house style.


