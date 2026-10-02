# Keymap & Configuration Spec — for porting to a Totem keyboard

This document fully describes the firmware configuration in this repo
(`zmk-urchin`, commit `292c34d "myconfig"`) so that the identical behavior can be
reimplemented as ZMK firmware for a **Totem** split keyboard.

Source of truth files:

| File | Content |
|---|---|
| `config/urchin.keymap` | All layers, combos, behaviors, macros |
| `config/urchin.conf` | Kconfig / power / BLE settings |
| `build.yaml` | CI build matrix |
| `config/boards/shields/urchin/*` | Vendor shield definition (hardware glue) |

`config/urchin.backup` is the **old** (QWERTY/Italian) firmware and is only
historical reference — do **not** port it.

---

## 1. Design summary (what you are porting)

* 5-layer split keymap, Seniply-inspired, Colemak-flavoured alphas.
* **OS keyboard layout = German (DE).** All raw ZMK keycodes are chosen so they
  render German characters (umlauts, `ß`, `§`, `€` …). ⚠️ The README's note
  about "Italian keyboard input" is a stale leftover from the original author's
  config — it does **not** apply to this keymap.
* Heavy use of:
  * mod-tap home-row modifiers (`&mt`),
  * key **combos** (chords) for Escape, German umlauts, caps-word, delete,
    backspace, Bluetooth layer access,
  * **mod-morph** behaviors for dual-purpose punctuation (e.g. `$`/`€`,
    `(`/`<`),
  * a custom hold-tap ("hyper space") that opens a Hyprland (window-manager)
    layer while holding `Super`.
* Power saving: deep sleep, 30 min idle timeout, keys wake the keyboard.

### Notation used throughout this document

> **Every layout table has two columns:**
> * **Types** — the character/function the keyboard actually produces with the
>   **German** OS layout active. This is what a user sees and is never the
>   keycode's US legend.
> * **Binding** — the raw ZMK keycode or behavior, copied verbatim from the
>   source. Do **not** read the binding as its US-ANSI legend: where keycode
>   and output differ (e.g. `&kp MINUS` types `ß`, `&kp SLASH` types `-`),
>   that is intentional. §8 explains the German keycode mapping.
>
> **Hold** — the function when the key is held instead of tapped.

---

## 2. Source hardware (Urchin) — position numbering

Combos are defined by **key positions** (matrix-transform order), not by key
codes, so the position grid must be understood and remapped.

Urchin: 34-key split (3×5 grid + 2 thumb keys per side), nice_nano_v2.
Position numbering derived from `config/boards/shields/urchin/urchin.dtsi`:

```text
Left half                                Right half
 0  1  2  3  4        |        5  6  7  8  9      top row
10 11 12 13 14        |       15 16 17 18 19      home row
20 21 22 23 24        |       25 26 27 28 29      bottom row
      30 31           |             32 33         thumbs
   outer inner         |         inner outer
```

Base layer, per position: character typed (German layout) — mod-tap keys
marked `*`:

```text
 0:Q  1:Y  2:O  3:P  4:Z     5:W  6:C  7:M  8:V  9:X
10:H* 11:A* 12:E* 13:I* 14:U  15:D 16:T* 17:N* 18:S* 19:R*
20:J  21:.  22:,  23:K  24:/  25:B  26:G  27:L  28:F  29:ß
```

Thumb positions: 30 = Tab / hold → Layer 1, 31 = Space / hold → Layer 4
("hyper"), 32 = Enter, 33 = `-` / hold → Layer 2.

---

## 3. Target hardware (Totem) — position numbering

Totem: **38-key** split (3×5 grid + 1 extra bottom-outer key + **3** thumb keys
per side), powered by a Seeed XIAO BLE (also nRF52840 — the `.conf` power
settings carry over).

⚠️ The `totem` shield is **not** in mainline ZMK — vendor the shield folder
from the canonical repo [GEIGEIGEIST/zmk-config-totem](https://github.com/GEIGEIGEIST/zmk-config-totem)
(`config/boards/shields/totem/*`) into this repo, exactly mirroring how the
`urchin` shield is vendored here.

Totem transform (verbatim from `totem.dtsi`):

```text
map = <
        RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4)    RC(0,5) RC(0,6) RC(0,7) RC(0,8) RC(0,9)
        RC(1,0) RC(1,1) RC(1,2) RC(1,3) RC(1,4)    RC(1,5) RC(1,6) RC(1,7) RC(1,8) RC(1,9)
RC(3,0) RC(2,0) RC(2,1) RC(2,2) RC(2,3) RC(2,4)    RC(2,5) RC(2,6) RC(2,7) RC(2,8) RC(2,9) RC(3,9)
                                RC(3,2) RC(3,3) RC(3,4)    RC(3,5) RC(3,6) RC(3,7)
>;
```

Resulting position numbers on Totem:

```text
Left half                                           Right half
 0  1  2  3  4        |       5  6  7  8  9          top row
10 11 12 13 14        |      15 16 17 18 19          home row
20 21 22 23 24 25     |     26 27 28 29 30 31        bottom row (20/31 = extra outer key)
      32 33 34        |           35 36 37           thumbs
   outer mid inner    |        inner mid outer
```

### Urchin → Totem physical key mapping

| Urchin pos | Physical key | → Totem pos |
|---|---|---|
| 0–4 | left top row | **0–4** (unchanged) |
| 5–9 | right top row | **5–9** (unchanged) |
| 10–14 | left home row | **10–14** (unchanged) |
| 15–19 | right home row | **15–19** (unchanged) |
| 20–24 | left bottom row | **21–25** |
| 25–29 | right bottom row | **26–30** |
| 30 | outer-left thumb (Tab / L1) | **32** |
| 31 | inner-left thumb (Space / hyper) | **33** (mid-left; alternative: 34) |
| 32 | inner-right thumb (Enter) | **36** (mid-right; alternative: 35) |
| 33 | outer-right thumb (`-` / L2) | **37** |

Free keys on Totem (Urchin has no equivalent): `20`, `31` (extra bottom-outer
keys) and `34`, `35` (innermost thumbs). Suggested use: `34` = Space again,
`35` = Enter again (thumb-roll comfort), `20`/`31` = `&none` or spare bindings.
Keep `34`/`35` consistent with whatever Space/Enter positions you chose
(they appear in the `fn_ble` combo, see §6).

---

## 4. Layers

Layer indices:

```c
#define BASE 0
#define OHSHORTS 1   // "ShortNavi"
#define SYM 2        // "SymNum"
// 3 = FnBle ("FuncBluetooth") — no #define, referenced as &mo 3
// 4 = Hypr — no #define, referenced by the hyper-space macro
```

### 4.1 Layer 0 — Base ("Base")

Left half (rows in position order):

| Pos | Types | Binding | Hold |
|---|---|---|---|
| 0 | `Q` | `&kp Q` | — |
| 1 | `Y` | `&kp Y` | — |
| 2 | `O` | `&kp O` | — |
| 3 | `P` | `&kp P` | — |
| 4 | `Z` | `&kp Z` | — |
| 10 | `H` | `&mt LCTRL H` | Left Ctrl |
| 11 | `A` | `&mt LALT A` | Left Alt |
| 12 | `E` | `&mt LGUI E` | Left GUI/Super |
| 13 | `I` | `&mt LSHIFT I` | Left Shift |
| 14 | `U` | `&kp U` | — |
| 20 | `J` | `&kp J` | — |
| 21 | `.` | `&kp PERIOD` | — |
| 22 | `,` | `&kp COMMA` | — |
| 23 | `K` | `&kp K` | — |
| 24 | `/` | `&sl_qu` (mod-morph, §7.1) | (with either Shift held: types `!`) |
| 30 | `Tab` | `&lt 1 TAB` | momentary Layer 1 |
| 31 | `Space` | `&hyper_space 0 SPACE` | Layer 4 + Super held (§7.2) |

Right half:

| Pos | Types | Binding | Hold |
|---|---|---|---|
| 5 | `W` | `&kp W` | — |
| 6 | `C` | `&kp C` | — |
| 7 | `M` | `&kp M` | — |
| 8 | `V` | `&kp V` | — |
| 9 | `X` | `&kp X` | — |
| 15 | `D` | `&kp D` | — |
| 16 | `T` | `&mt LSHIFT T` | Left Shift |
| 17 | `N` | `&mt RGUI N` | Right GUI/Super |
| 18 | `S` | `&mt RALT S` | Right Alt (AltGr) |
| 19 | `R` | `&mt RCTRL R` | Right Ctrl |
| 25 | `B` | `&kp B` | — |
| 26 | `G` | `&kp G` | — |
| 27 | `L` | `&kp L` | — |
| 28 | `F` | `&kp F` | — |
| 29 | `ß` | `&kp MINUS` | — |
| 32 | `Enter` | `&kp RETURN` | — |
| 33 | `-` | `&lt 2 SLASH` | momentary Layer 2 |

### 4.2 Layer 1 — ShortNavi ("Shorts + navigation"), one-handed shortcuts

Momentary via left-outer thumb (Tab position), layer index 1.
All shortcuts are layout-independent.

Left half:

| Pos | Types | Binding |
|---|---|---|
| 0 | `Ctrl`+`Q` | `&kp LC(Q)` |
| 1 | `Ctrl`+`W` | `&kp LC(W)` |
| 2 | `Ctrl`+`E` | `&kp LC(E)` |
| 3 | `Ctrl`+`R` | `&kp LC(R)` |
| 4 | `Ctrl`+`T` | `&kp LC(T)` |
| 10 | `Ctrl`+`A` | `&kp LC(A)` |
| 11 | `Ctrl`+`S` | `&kp LC(S)` |
| 12 | `Ctrl`+`D` | `&kp LC(D)` |
| 13 | `Ctrl`+`F` | `&kp LC(F)` |
| 14 | `Ctrl`+`G` | `&kp LC(G)` |
| 20 | `Ctrl`+`Y` (redo — note: no Ctrl+Z on this layer) | `&kp LC(Y)` |
| 21 | `Ctrl`+`X` | `&kp LC(X)` |
| 22 | `Ctrl`+`C` | `&kp LC(C)` |
| 23 | `Ctrl`+`V` | `&kp LC(V)` |
| 24 | `Ctrl`+`B` | `&kp LC(B)` |
| 32 | (transparent) | `&trans` |

Right half:

| Pos | Types | Binding |
|---|---|---|
| 5 | `PageUp` | `&kp PAGE_UP` |
| 6 | `Home` | `&kp HOME` |
| 7 | `↑` | `&kp UP` |
| 8 | `End` | `&kp END` |
| 9 | (none) | `&none` |
| 15 | `PageDown` | `&kp PAGE_DOWN` |
| 16 | `←` | `&kp LEFT` |
| 17 | `↓` | `&kp DOWN` |
| 18 | `→` | `&kp RIGHT` |
| 19 | `Left Shift` (plain, non-hold) | `&kp LEFT_SHIFT` |
| 25 | `PrintScreen` | `&kp PRINTSCREEN` |
| 26 | `1` | `&kp N1` |
| 27 | `2` | `&kp N2` |
| 28 | `3` | `&kp N3` |
| 29 | (none) | `&none` |
| 33, 36, 37 (Totem thumbs) | (none) | `&none` (source: `&none &none &none` on positions 31/32/33) |

### 4.3 Layer 2 — SymNum ("SymNum"), symbols & numbers

Momentary via right-outer thumb, layer index 2.
For mod-morph keys the second Types column is the output **with Shift held**.

Left half:

| Pos | Types | with Shift | Binding |
|---|---|---|---|
| 0 | `´` (dead acute) | `` ` `` (dead grave) | `&ticks` (§7.1) |
| 1 | `$` | `€` | `&eur_usd` (§7.1) |
| 2 | `'` | `"` | `&quotes` (§7.1) |
| 3 | `[` | — | `&kp RA(N8)` |
| 4 | `]` | — | `&kp RA(N9)` |
| 10 | `~` | `^` | `&tilde_circumflex` (§7.1) |
| 11 | `\` | — | `&kp RA(MINUS)` |
| 12 | `@` | `#` | `&at_hash` (§7.1) |
| 13 | `(` | `<` | `&lesser_bracket` (§7.1) |
| 14 | `)` | `>` | `&bigger_bracket` (§7.1) |
| 20 | `&` | `§` | `&and_law` (§7.1) |
| 21 | `%` | — | `&kp LS(N5)` |
| 22 | `\|` | — | `&kp RA(NON_US_BSLH)` |
| 23 | `{` | — | `&kp RA(N7)` |
| 24 | `}` | — | `&kp RA(N0)` |
| 32 (Totem thumb) | `Left Shift` (plain, non-hold) | | `&kp LEFT_SHIFT` |

Right half:

| Pos | Types | with Shift | Binding |
|---|---|---|---|
| 5 | `.` | — | `&kp PERIOD` |
| 6 | `7` | — | `&kp N7` |
| 7 | `8` | — | `&kp N8` |
| 8 | `9` | — | `&kp N9` |
| 9 | `+` | `*` | `&add_mult` (§7.1) |
| 15 | `0` | — | `&kp N0` |
| 16 | `4` | — | `&mt LEFT_SHIFT N4` (hold: Left Shift) |
| 17 | `5` | — | `&kp N5` |
| 18 | `6` | — | `&kp N6` |
| 19 | `-` | `/` | `&sub_div` (§7.1) |
| 25 | `,` | — | `&kp COMMA` |
| 26 | `1` | — | `&kp N1` |
| 27 | `2` | — | `&kp N2` |
| 28 | `3` | — | `&kp N3` |
| 29 | `=` | — | `&kp LS(N0)` |
| 33 / 36 (Totem thumbs) | (none) | | `&none` |
| 37 (Totem thumb) | (transparent) | | `&trans` |

(Source thumb bindings: position 30 = `&kp LEFT_SHIFT`, 31 = `&none`,
32 = `&none`, 33 = `&trans`.)

### 4.4 Layer 3 — FnBle ("FuncBluetooth"), function keys & Bluetooth

Momentary via the **combo holding both inner thumbs (Space+Enter)** — see §6.
No `layers` restriction: the combo works from any layer.

Left half:

| Pos | Types | Binding |
|---|---|---|
| 0–4 | (none) | `&none` |
| 10 | Select Bluetooth profile 4 | `&bt BT_SEL 4` |
| 11 | Select Bluetooth profile 3 | `&bt BT_SEL 3` |
| 12 | Select Bluetooth profile 2 | `&bt BT_SEL 2` |
| 13 | Select Bluetooth profile 1 | `&bt BT_SEL 1` |
| 14 | Select Bluetooth profile 0 | `&bt BT_SEL 0` |
| 20–24 | (none) | `&none` |
| 32 (Totem thumb) | (none) | `&none` |

Right half:

| Pos | Types | Binding |
|---|---|---|
| 5 | (none) | `&none` |
| 6 | `F7` | `&kp F7` |
| 7 | `F8` | `&kp F8` |
| 8 | `F9` | `&kp F9` |
| 9 | `F10` | `&kp F10` |
| 15 | (none) | `&none` |
| 16 | `F4` | `&kp F4` |
| 17 | `F5` | `&kp F5` |
| 18 | `F6` | `&kp F6` |
| 19 | `F11` | `&kp F11` |
| 25 | (none) | `&none` |
| 26 | `F1` | `&kp F1` |
| 27 | `F2` | `&kp F2` |
| 28 | `F3` | `&kp F3` |
| 29 | `F12` | `&kp F12` |
| 33 / 36 (Totem thumbs) | (transparent — the held Space/Enter thumb must not fire) | `&trans` |
| 37 (Totem thumb) | (none) | `&none` |

(Source thumb bindings: 30 = `&none`, 31 = `&trans`, 32 = `&trans`,
33 = `&none`.)

On this layer, the **del_ble combo** (press the four left home-row keys H, A,
E, I together, see §6) clears the active Bluetooth profile bond
(`&bt BT_CLR`). BT profiles 0–4 are selected with the left home row.

### 4.5 Layer 4 — Hypr (Hyprland / window-manager layer)

Activated by **holding Space** (the hyper-space hold-tap). While the layer is
active, `Super` (`LGUI`) is **already held** by the hyper-space macro — so the
actual chords sent are Super+X etc. even though the bindings write plain keys.

Convention on this layer (`&mt` with full keypress as hold behavior):
*tap* = window command, *hold* = same command with extra `Shift` (i.e.
move/resize variant).

Left half:

| Pos | Types (tap) | Types (hold) | Binding |
|---|---|---|---|
| 0 | `Super`+`H` | `Super`+`Shift`+`H` | `&mt LG(LS(H)) LG(H)` |
| 1 | `Super`+`J` | `Super`+`Shift`+`J` | `&mt LG(LS(J)) LG(J)` |
| 2 | `Super`+`K` | `Super`+`Shift`+`K` | `&mt LG(LS(K)) LG(K)` |
| 3 | `Super`+`L` | `Super`+`Shift`+`L` | `&mt LG(LS(L)) LG(L)` |
| 4 | `Super`+`D` (launcher) | `Super`+`Enter` (terminal) | `&mt LG(ENTER) LG(D)` |
| 10 | `Super`+`1` | `Super`+`Shift`+`1` | `&mt LS(LG(N1)) LG(N1)` |
| 11 | `Super`+`2` | `Super`+`Shift`+`2` | `&mt LG(LS(N2)) LG(N2)` |
| 12 | `Super`+`3` | `Super`+`Shift`+`3` | `&mt LG(LS(N3)) LG(N3)` |
| 13 | `Super`+`4` | `Super`+`Shift`+`4` | `&mt LG(LS(N4)) LG(N4)` |
| 14 | `Super`+`5` | `Super`+`Shift`+`5` | `&mt LG(LS(N5)) LG(N5)` |
| 20 | `Space` (carries held Super → `Super`+`Space`) | `Super`+`Shift`+`Q` | `&mt LG(LS(Q)) SPACE` |
| 21 | `Super`+`Space` | — | `&kp LG(SPACE)` |
| 22 | `Super`+`Backspace` | — | `&kp LG(BACKSPACE)` |
| 23 | `Super`+`F` | — | `&kp LG(F)` |
| 24 | `Super`+`Q` (quit) | — | `&kp LG(Q)` |

Right half and remaining thumbs: all `&none`, except the held-space thumb
(`31`, → Totem `33`) which is `&trans` (its `&hyper_space` behavior continues
to work).

---

## 5. Global behaviors

```dts
&mt {
    flavor = "balanced";
};
```
All mod-taps use **balanced** flavor (favor whichever press came first when
rolling tap→hold / hold→tap).

---

## 6. Combos (all timeouts in ms)

⚠️ Combos are **position-based**. The "Keys" column names the physical keys to
chord (letters as printed on the keycaps — identical on US and German layouts);
the "→ Totem pos" column applies the §3 mapping.

| Name | Output (Types) | Keys | Timeout | Active layers | Urchin pos | Totem pos |
|---|---|---|---|---|---|---|
| `combo_escape` | `Escape` | Y + O | 70 | 0 | 1 2 | 1 2 |
| `combo_auml` | `ä` | A + E | 70 | 0 | 11 12 | 11 12 |
| `combo_ouml` | `ö` | O + E | 70 | 0 | 2 12 | 2 12 |
| `combo_uuml` | `ü` | E + U | 70 | 0 | 12 14 | 12 14 |
| `combo_ei` | literal string `ei` (`&ei_key` macro) | E + I | 70 | 0 | 12 13 | 12 13 |
| `caps_word` | Caps Word (`&caps_word`) | U + D | **50** | 0 | 14 15 | 14 15 |
| `fn_ble` | momentary Layer 3 (`&mo 3`) | **Space + Enter (both inner thumbs)** | **60** | **all layers** | 31 32 | **33 36** |
| `del_ble` | clear BT bond (`&bt BT_CLR`) | H + A + E + I (four left home-row keys) | **100** | 3 | 10 11 12 13 | 10 11 12 13 |
| `backspace` | `Backspace` | M + V | 70 | 0 **and** 2 | 8 7 | 8 7 |
| `delete` | `Delete` | K + `,` | 70 | 0 | 23 22 | **24 23** |

Notes:
* The `// both right thumb keys` comment above `combo_escape` in the source is
  **stale/wrong** — it is really the Y+O chord.
* `combo_ei`'s purpose is unclear (types the literal string "ei"); port it
  as-is.
* `backspace` is explicitly enabled on layers `<0 2>`; `delete` only on layer
  0.

---

## 7. Custom behaviors & macros

### 7.1 Mod-morphs

All trigger on `MOD_LSFT` (either Shift key), except `sl_qu` which triggers on
`MOD_LSFT|MOD_RSFT` (same effect — both Shift keys).

| Name | Types (normal) | Types (Shift held) | Binding | Used at |
|---|---|---|---|---|
| `sl_qu` | `/` | `!` | `<&kp LS(N7)>, <&kp LS(N1)>` | Base pos 24 |
| `add_mult` | `+` | `*` | `<&kp RIGHT_BRACKET>, <&kp LS(RIGHT_BRACKET)>` | Sym pos 9 |
| `sub_div` | `-` | `/` | `<&kp SLASH>, <&kp LS(N7)>` | Sym pos 19 |
| `eur_usd` | `$` | `€` | `<&kp LS(N4)>, <&kp RA(E)>` | Sym pos 1 |
| `tilde_circumflex` | `~` | `^` | `<&kp RA(RIGHT_BRACKET)>, <&kp GRAVE>` | Sym pos 10 |
| `ticks` | `´` | `` ` `` | `<&kp EQUAL>, <&kp LS(EQUAL)>` | Sym pos 0 |
| `quotes` | `'` | `"` | `<&kp LS(BACKSLASH)>, <&kp LS(N2)>` | Sym pos 2 |
| `lesser_bracket` | `(` | `<` | `<&kp LS(N8)>, <&kp NON_US_BSLH>` | Sym pos 13 |
| `bigger_bracket` | `)` | `>` | `<&kp LS(N9)>, <&kp LS(NON_US_BSLH)>` | Sym pos 14 |
| `and_law` | `&` | `§` | `<&kp LS(N6)>, <&kp LS(N3)>` | Sym pos 20 |
| `at_hash` | `@` | `#` | `<&kp RA(Q)>, <&kp NUHS>` | Sym pos 12 |
| `new_behavior` | — **dead code, not referenced by any layer** | | `<&kp LS(N8)>, <&kp LESS_THAN>` | — |

### 7.2 Hold-tap: `hyper_space`

```dts
hyper_space: hyper_space {
    compatible = "zmk,behavior-hold-tap";
    label = "HYPER_SPACE";
    bindings = <&hyper_gui>, <&kp>;
    flavor = "hold-preferred";
    #binding-cells = <2>;
    tapping-term-ms = <200>;
    quick-tap-ms = <400>;
};
```
* Tap → second binding (`&kp SPACE`).
* Hold → first binding (`&hyper_gui` macro): activates **layer 4** and holds
  `LGUI` for as long as the key is held (`hold-preferred`, 200 ms
  tapping-term, 400 ms quick-tap so fast repeated taps stay taps).

### 7.3 Macros

```dts
hyper_gui: hyper_gui {
    compatible = "zmk,behavior-macro";
    #binding-cells = <0>;
    bindings =
        <&macro_press &mo 4 &kp LGUI>,
        <&macro_pause_for_release>,
        <&macro_release &mo 4 &kp LGUI>;
};

ei_key: ei_key {
    compatible = "zmk,behavior-macro";
    #binding-cells = <0>;
    bindings = <&kp E &kp I>;
};
```

---

## 8. German layout cheat-sheet (why the raw codes render these chars)

The Types column in this document is derived from the raw bindings via the
German OS layout. This table is the mapping (keycodes written in their
US-ANSI legend — **this section is the only place where US legends are used**):

| ZMK keycode(s) (US legend) | German layout output |
|---|---|
| `APOSTROPHE` (0x34) / `SEMICOLON` (0x33) / `LEFT_BRACKET` (0x2F) | `ä` / `ö` / `ü` |
| `MINUS` (0x2D) / `SLASH` (0x38) | `ß` / `-` |
| `EQUAL` (0x2E) / `LS(EQUAL)` | `´` (dead) / `` ` `` (dead) |
| `GRAVE` (0x35) / `RIGHT_BRACKET` (0x30) | `^` / `+` |
| `BACKSLASH` (0x31) / `NUHS` (0x32) | `#`-key (`'` shifted) / `#` |
| `NON_US_BSLH` (0x64) | `<` (`>` shifted, `\|` via AltGr) |
| `N1…N0` shifted | `!` `"` `§` `$` `%` `&` `/` `(` `)` `=` |
| `RA(E)` / `RA(Q)` / `RA(MINUS)` / `RA(RIGHT_BRACKET)` / `RA(N7..N0)` / `RA(NON_US_BSLH)` | `€` `@` `\` `~` `{ [ ] }` `\|` |

⚠️ The exact rendering of the two ISO extra keys (`BACKSLASH` 0x31 vs
`NUHS` 0x32) varies slightly between OSes; the author uses 0x31 for `'` and
0x32 for `#`. Do **not** "clean these up" when porting — keep the raw codes.

---

## 9. `config/urchin.conf` (Kconfig settings to carry over)

```ini
# increase bluetooth signal power
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y

# enable deep sleep support
CONFIG_ZMK_SLEEP=y

# sleep after 30 minutes of idle (1,800,000 ms);
# reconnect may take a few seconds after waking
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=1800000
```

Commented-out (inactive, keep or drop as desired): `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN`,
eager debouncing (`CONFIG_ZMK_KSCAN_DEBOUNCE_PRESS_MS=1` /
`CONFIG_ZMK_KSCAN_DEBOUNCE_RELEASE_MS=5`), WPM-widget off, battery percentage.

Also in the shield file (`urchin.dtsi`): the kscan node has
`wakeup-source;` so **any keypress wakes the keyboard from deep sleep** —
make sure the Totem shield's kscan gets this too.

---

## 10. Build configuration

Current (`build.yaml`): `nice_nano_v2` + `urchin_left` / `urchin_right`
(nice_view displays **disabled** in the matrix, commented out) +
`settings_reset`.

For Totem (`config/totem.conf`, `config/totem.keymap`, board = Seeed XIAO BLE):

```yaml
include:
  - board: seeeduino_xiao_ble
    shield: totem_left
  - board: seeeduino_xiao_ble
    shield: totem_right
```
Note: there is no `settings_reset` build for the XIAO (per the canonical Totem
repo); the urchin builds can be kept alongside for the old hardware.

Steps mirroring this repo's structure:

1. Vendor the shield: copy `config/boards/shields/totem/*` from
   [GEIGEIGEIST/zmk-config-totem](https://github.com/GEIGEIGEIST/zmk-config-totem)
   into this repo (Totem is **not** in mainline ZMK; this repo's `west.yml`
   tracks `zmkfirmware/zmk` `main`).
2. Replace the vendored `config/boards/shields/totem/totem.keymap` with a stub
   that includes the user keymap, same pattern as here:
   ```c
   #include "../../../totem.keymap"
   ```
3. Put the ported keymap in `config/totem.keymap` and the Kconfig settings in
   `config/totem.conf` (name matching the shield is what makes Zephyr pick
   them up).
4. Update `build.yaml` as above; the existing `.github/workflows/build.yml`
   needs no changes.

---

## 11. Porting checklist (gotchas for the implementing LLM)

1. **Position shift**: bottom rows move (Urchin 20–24/25–29 → Totem
   21–25/26–30) and thumbs move (30/31/32/33 → 32/33/36/37). Top and home
   rows (positions 0–19) are identical on both boards.
2. **Layer binding arrays must be re-expanded to 38 entries** in Totem
   transform order: 10 (top) + 10 (home) + 12 (bottom — starts with the extra
   key at position 20!) + 6 (thumbs: 32,33,34,35,36,37).
3. **Combos**: only `delete` (23 22 → 24 23) and `fn_ble` (31 32 → 33 36, or
   34 35 if Space/Enter land on the innermost thumbs) change positionally;
   all others transfer 1:1 because they sit on top/home-row chords. Decide
   the Space/Enter thumb positions **first**, then set `fn_ble` to match.
4. Keep `&hyper_space 0 SPACE` on whatever thumb position Space moves to, and
   keep that position `&trans` in layers 3 and 4 (and the Enter thumb `&trans`
   in layer 3).
5. Keep all raw keycodes verbatim (they are German-layout dependent) — see §8.
6. Add `wakeup-source;` to the Totem kscan node (the Totem shield's dtsi
   doesn't have it).
7. Update the stale README note: OS layout is **German**, not Italian.
8. Optionally clean up: `new_behavior` (unused mod-morph), the stale
   "both right thumb keys" comment on `combo_escape`, and decide whether
   `combo_ei` (types literal "ei") is still wanted.
9. XIAO BLE has fewer onboard resources than nice_nano_v2 — keep the Kconfig
   settings minimal as in §9.

---

## References

* [GEIGEIGEIST/zmk-config-totem](https://github.com/GEIGEIGEIST/zmk-config-totem) — canonical Totem shield definition, transform and build matrix
* [ZMK documentation](https://zmk.dev/docs) — behaviors, combos, mod-morphs, macros
* [ZMK keycode list](https://zmk.dev/docs/codes)
* [Seniply layout](https://stevep99.github.io/seniply/) — design inspiration for this keymap
