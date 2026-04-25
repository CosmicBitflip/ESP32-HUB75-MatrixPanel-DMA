# Single-Scan 32×64 Panel Support — Change Catalogue
**Repository:** CosmicBitflip/ESP32-HUB75-MatrixPanel-DMA  
**Branch:** `feat/single-scan-support`  
**Base branch:** `master`  
**Commits unique to this branch:**
- `503c9d9` – WIP matrix_row_in_parallel needs to be fixed  
- `f926bef` – first draft single_scan_mode  
- `44d200d` – fixed VirtualMatrixPanel_T  
- `858398d` – Add comprehensive documentation for SINGLE_SCAN_SUPPORT_CHANGES  
- `b901b37` – Update SINGLE_SCAN_SUPPORT_CHANGES.md with comprehensive documentation  

**Files changed:**
1. `src/ESP32-HUB75-MatrixPanel-I2S-DMA.h`  
2. `src/ESP32-HUB75-MatrixPanel-I2S-DMA.cpp`  
3. `src/ESP32-HUB75-VirtualMatrixPanel_T.hpp`  
4. `SINGLE_SCAN_SUPPORT_CHANGES.md` *(new file — project-level documentation)*  

---

## Background / Motivation

Standard HUB75 LED panels use a **1/16-scan** drive scheme: two rows of pixels are driven simultaneously through two independent RGB channel pairs (RGB1 and RGB2), so `MATRIX_ROWS_IN_PARALLEL = 2` and `ROWS_PER_FRAME = panel_height / 2 = 16`.

Some 32×64 (and other) panels use a **1/32-scan** (single-scan) drive scheme: only one row is driven at a time through a single RGB channel pair.  This requires:
- One fewer RGB channel pair (R2/G2/B2 pins unused / set to -1).
- The full set of address lines A–E (5 bits → 32 rows).
- `ROWS_PER_FRAME = panel_height = 32` (no halving).
- `matrix_rows_in_parallel = 1` (not 2).

All changes in this branch implement, validate, and document this mode.

---

## 1. `src/ESP32-HUB75-MatrixPanel-I2S-DMA.h`

### 1.1 Removed compile-time `MATRIX_ROWS_IN_PARALLEL` macro, replaced with a runtime variable

**Before:**
```cpp
#ifdef MATRIX_ROWS_IN_PARALLEL
#pragma message "You are not supposed to set MATRIX_ROWS_IN_PARALLEL. Setting it back to default."
#undef MATRIX_ROWS_IN_PARALLEL
#endif
#define MATRIX_ROWS_IN_PARALLEL 2
```

**After:**
The block is commented out entirely.  
A new **private instance variable** is declared instead:
```cpp
uint8_t matrix_rows_in_parallel; // Replace matrix_rows_in_parallel macro
```

**Why:** A compile-time constant `= 2` cannot vary between panels. Making it a runtime value lets a single firmware binary support both 1/16-scan and 1/32-scan panels selected through configuration.

---

### 1.2 New `single_scan` field in `HUB75_I2S_CFG`

```cpp
// single scan addition
bool single_scan = false;
```

Added to the `HUB75_I2S_CFG` struct. Defaults to `false` so all existing code compiles and runs identically when the field is not touched.

---

### 1.3 Constructor parameter `_single_scan` added to `HUB75_I2S_CFG`

**Before:**
```cpp
HUB75_I2S_CFG(..., uint8_t _pixel_color_depth_bits = PIXEL_COLOR_DEPTH_BITS_DEFAULT)
  : mx_width(_w), mx_height(_h), ..., min_refresh_rate(_min_refresh_rate)
```

**After:**
```cpp
HUB75_I2S_CFG(...,
    uint8_t _pixel_color_depth_bits = PIXEL_COLOR_DEPTH_BITS_DEFAULT,
    bool _single_scan = false)
  : mx_width(_w), mx_height(_h), ..., min_refresh_rate(_min_refresh_rate),
    single_scan(_single_scan)
```

Two changes:
1. `_single_scan` defaulting to `false` appended as the last parameter.
2. The constructor initializer list now also passes `line_decoder(_line_drv)` (which was silently missing from the base branch).

---

### 1.4 Constructors of `MatrixPanel_I2S_DMA` now initialize RGB bitmasks eagerly

Both the default constructor and the `opts`-accepting constructor now call:
```cpp
rgb1_clear_mask  = 0b1111111111111000;  // Default RGB1 mask
rgb2_clear_mask  = 0b1111111111000111;  // Default RGB2 mask
rgb12_clear_mask = 0b1111111111000000;  // Default RGB1+RGB2 mask
```

**Why:** The three mask variables were added as plain (uninitialized) private members. Setting them in every constructor ensures they always have valid values before any pixel writes occur.

---

### 1.5 `begin()` — validate single-scan pin constraints

Inside `begin()`, immediately after logging the GPIO assignments:
```cpp
if (m_cfg.single_scan) {
    if (m_cfg.gpio.r2 >= 0 || m_cfg.gpio.g2 >= 0 || m_cfg.gpio.b2 >= 0) {
        ESP_LOGE("begin()", "R2/G2/B2 must be -1 for single-scan");
        return false;
    }
    if (m_cfg.mx_height > 16 && m_cfg.gpio.e < 0) {
        ESP_LOGE("begin()", "E_PIN required for single-scan with height >16");
        return false;
    }
}
```

**Why:** Single-scan panels must not have R2/G2/B2 connected (only one RGB group is used), and panels taller than 16 rows need the E address pin to address all 32 rows. Returning early with a clear error message prevents silent corruption.

---

### 1.6 `begin()` — call `initBitmasks()` after DMA allocation

```cpp
initBitmasks(); // Initialize bitmasks based on single_scan
```

Called after the DMA frame buffer is successfully allocated, to allow `initBitmasks()` to run at the correct point in the lifecycle.

---

### 1.7 `setCfg()` — runtime `ROWS_PER_FRAME` and `matrix_rows_in_parallel` calculation

**Before:**
```cpp
PIXELS_PER_ROW = m_cfg.mx_width * m_cfg.chain_length;
ROWS_PER_FRAME = m_cfg.mx_height / MATRIX_ROWS_IN_PARALLEL;
```

**After:**
```cpp
if (m_cfg.single_scan) {
    matrix_rows_in_parallel = 1;
    if (m_cfg.mx_height == 32 && m_cfg.gpio.e == -1) {
        ESP_LOGE("setCfg", "E pin required for single_scan on 32-height panel");
        return false;
    }
    ROWS_PER_FRAME = m_cfg.mx_height; // 32 rows for 1/32 scan
} else {
    matrix_rows_in_parallel = 2;
    ROWS_PER_FRAME = m_cfg.mx_height / matrix_rows_in_parallel; // Default 16
}
PIXELS_PER_ROW = m_cfg.mx_width * m_cfg.chain_length;
```

**Why:** `ROWS_PER_FRAME` drives how many DMA row-buffers are allocated and how pixel data is indexed. For single-scan, all rows must be addressable independently; halving the height would skip half the rows.

---

### 1.8 Private member variables changed from in-class initializers to uninitialized declarations

**Before:**
```cpp
uint16_t PIXELS_PER_ROW = m_cfg.mx_width * m_cfg.chain_length;
uint8_t  ROWS_PER_FRAME = m_cfg.mx_height / MATRIX_ROWS_IN_PARALLEL;
uint8_t  MASK_OFFSET    = 16 - m_cfg.getPixelColorDepthBits();
```

**After (commented out, then redeclared):**
```cpp
uint16_t PIXELS_PER_ROW;
uint8_t  ROWS_PER_FRAME;
uint8_t  MASK_OFFSET;
```

**Why:** The in-class initializers ran at object construction time, before `setCfg()` could set the correct values for single-scan. Removing the initializers forces all assignments to go through `setCfg()`, which now contains the mode-aware logic.

---

### 1.9 New private member variables and method for bitmask management

```cpp
uint16_t rgb1_clear_mask;
uint16_t rgb2_clear_mask;
uint16_t rgb12_clear_mask;
void initBitmasks();
```

And the inline implementation in the header:
```cpp
inline void MatrixPanel_I2S_DMA::initBitmasks() {
    rgb2_clear_mask  = 0b1111111111000111;
    rgb12_clear_mask = 0b1111111111000000;
}
```

**Why:** The existing code used the compile-time macros `BITMASK_RGB1_CLEAR`, `BITMASK_RGB2_CLEAR`, `BITMASK_RGB12_CLEAR` directly. Making them runtime variables allows `initBitmasks()` to be extended in the future to set mode-specific masks (e.g., to disable the RGB2 bits entirely in single-scan mode).

---

## 2. `src/ESP32-HUB75-MatrixPanel-I2S-DMA.cpp`

### 2.1 `setupDMA()` — duplicate single-scan row/parallel calculation

At the top of `setupDMA()`, before allocating frame buffers:
```cpp
if (m_cfg.single_scan) {
    matrix_rows_in_parallel = 1;
    if (m_cfg.mx_height == 32 && m_cfg.gpio.e == -1) {
        ESP_LOGE("I2S-DMA", "E pin required for single_scan on 32-height panel");
        return false;
    }
    ROWS_PER_FRAME = m_cfg.mx_height;
} else {
    matrix_rows_in_parallel = 2;
    ROWS_PER_FRAME = m_cfg.mx_height / matrix_rows_in_parallel;
}
```

This mirrors the logic added to `setCfg()` (change 1.7 above), ensuring `ROWS_PER_FRAME` is set correctly before the DMA buffer allocation that immediately follows.

---

### 2.2 `updateMatrixDMABuffer(x, y, r, g, b)` — single-scan pixel-write path

This is the core per-pixel drawing function.

**Original logic (dual-scan):**  
- If `y_coord >= ROWS_PER_FRAME`, the pixel belongs to the bottom half → use RGB2 bit positions and subtract `ROWS_PER_FRAME` from y to find the DMA row index.  
- Otherwise, use RGB1 bit positions.

**New logic:**
```cpp
if (m_cfg.single_scan) {
    _colourbitclear = BITMASK_RGB1_CLEAR; // Only clear RGB1
    // Address 0-31 using A-E
    uint16_t address_bits = 0;
    if (y_coord & 0x01) address_bits |= BIT_A;
    if (y_coord & 0x02) address_bits |= BIT_B;
    if (y_coord & 0x04) address_bits |= BIT_C;
    if (y_coord & 0x08) address_bits |= BIT_D;
    if (y_coord & 0x10) address_bits |= BIT_E;
    ESP32_I2S_DMA_STORAGE_TYPE *p = getRowDataPtr(y_coord, 0);
    p[x_coord] &= _colourbitclear;
} else {
    if (y_coord >= ROWS_PER_FRAME) {
        _colourbitoffset = BITS_RGB2_OFFSET;
        _colourbitclear  = BITMASK_RGB2_CLEAR;
        y_coord         -= ROWS_PER_FRAME;
    }
}
```

**Why:**  
Single-scan panels expose all rows individually (no halving). All data is written to the RGB1 bit positions; the RGB2 positions are never written. The A–E address bits (5 bits) allow addressing all 32 rows directly. The address calculation block (though it stores to a local variable not yet used in the actual DMA word write — a known work-in-progress note) demonstrates the bit-per-row addressing intent.

---

### 2.3 `updateMatrixDMABuffer(r, g, b)` (full-frame fill) — single-scan fill path

For the full-frame fill variant:

**Before:**
```cpp
// Duplicate bits for RGB2
RGB_output_bits |= RGB_output_bits << BITS_RGB2_OFFSET; // BGRBGR
```

**After:**
```cpp
if (m_cfg.single_scan) {
    RGB_output_bits <<= 0; // Only RGB1, no shift
} else {
    RGB_output_bits |= RGB_output_bits << BITS_RGB2_OFFSET; // Duplicate for RGB2
}
```

**Why:**  
In dual-scan mode both RGB1 and RGB2 bit groups need to be filled because both halves of the panel are driven simultaneously. In single-scan mode only the RGB1 group is active, so duplicating the bits into RGB2 positions would corrupt the address/OE bits that share the same word.

---

### 2.4 `clearFrameBuffer()` — single-scan address-bit writing

When writing row address bits (ABCDE) into the DMA buffer during a frame-buffer clear:

**Before:**  
All variants (SM5266P, SM5368, default) were handled in one unified loop.

**After:**
```cpp
if (m_cfg.single_scan) {
    abcde <<= BITS_ADDR_OFFSET;
    x_pixel = fb->rowBits[row_idx]->width * fb->rowBits[row_idx]->colour_depth;
    do {
        --x_pixel;
        row[ESP32_TX_FIFO_POSITION_ADJUST(x_pixel)] = abcde;
    } while (x_pixel != fb->rowBits[row_idx]->width);
} else {
    // original SM5266P / SM5368 / default handling preserved unchanged
    ...
}
```

**Why:**  
For single-scan mode the row index maps 1:1 to the ABCDE address lines without any special decoder chip logic, so the SM5266P and SM5368 branches are irrelevant. Using a dedicated branch keeps the single-scan path clean and avoids any unintended side-effects from driver-specific address masking.

---

### 2.5 `setBrightnessOE()` — re-indentation only (no logic change)

The entire `setBrightnessOE()` function was re-indented from 2-space to 4-space indentation and nested `do/while` loops were reformatted. **No logic was changed.** This change is documented here for completeness but is explicitly excluded from the functional change catalogue per the problem statement.

---

## 3. `src/ESP32-HUB75-VirtualMatrixPanel_T.hpp`

### 3.1 New `SINGLE_SCAN_32PX_HIGH` enum value

```cpp
SINGLE_SCAN_32PX_HIGH  ///< Single-scan mode, 32-pixel high panels with single RGB channel.
```

Added to the `PANEL_SCAN_TYPE` enum. This scan-type token is used as the template parameter to `ScanTypeMapping<>` and `VirtualMatrixPanel_T<>`.

---

### 3.2 `ScanTypeMapping<>` — `kScanType` compile-time constant

```cpp
static constexpr PANEL_SCAN_TYPE kScanType = ScanType;
```

Added as a public static constexpr member of `ScanTypeMapping<ScanType>`. Enables `if constexpr` guards elsewhere in the class hierarchy to inspect which scan type is active.

---

### 3.3 `ScanTypeMapping<>::apply()` — coordinate mapping for `SINGLE_SCAN_32PX_HIGH`

```cpp
else if constexpr (ScanType == SINGLE_SCAN_32PX_HIGH) {
    // Direct mapping for 1/32 scan, single RGB, 32 rows addressed via A-E
    coords.y = coords.y % 32; // Ensure y is 0-31
    return coords;
}
```

Added to the `apply()` function's chain of scan-type conditionals.

**Why:**  
Other scan types (FOUR_SCAN_32PX_HIGH, etc.) remap virtual pixel coordinates to account for the wiring topology of multi-scan panels. Single-scan panels drive rows sequentially; there is no topology remapping — the virtual y coordinate maps directly to the physical row address, so modulo-32 is sufficient to clamp the coordinate and the function returns immediately.

---

### 3.4 `drawDisplayTest()` — panel-numbering for single-scan

```cpp
if constexpr (ScanTypeMapping::ScanType == SINGLE_SCAN_32PX_HIGH) {
    this->print(panel_id); // Simpler numbering for single-scan
} else {
    this->print(vmodule_cols * vmodule_rows - panel_id + 1); // Default numbering
}
```

**Why:**  
The default display test labels panels in reverse order (`vmodule_cols * vmodule_rows - panel_id + 1`). For single-scan panels the forward sequential numbering is more intuitive and matches the physical layout.

---

### 3.5 `setDisplay()` — runtime validation for `SINGLE_SCAN_32PX_HIGH`

```cpp
if constexpr (ScanTypeMapping::kScanType == SINGLE_SCAN_32PX_HIGH) {
    if (panel_res_y != 32) {
        Serial.println("SINGLE_SCAN_32PX_HIGH requires panel height of 32");
    }
    const HUB75_I2S_CFG &cfg = display->getCfg();
    if (!cfg.single_scan || cfg.gpio.e == -1) {
        Serial.println("SINGLE_SCAN_32PX_HIGH requires single_scan=true and E pin");
    }
}
```

Added inside `setDisplay()`. Checks that:
1. The panel vertical resolution is 32.
2. The underlying `MatrixPanel_I2S_DMA` configuration has `single_scan = true`.
3. The E address pin is configured (not -1).

**Why:**  
This cross-checks that the virtual panel's compile-time template scan type is consistent with the runtime hardware configuration. Without this check, a mismatch between the template parameter and the HUB75 config struct would produce wrong output without any diagnostic.

---

## 4. `SINGLE_SCAN_SUPPORT_CHANGES.md` (new file)

A new project-level documentation file was added. Its content describes the feature at a high level and includes usage examples and summary tables. The actual code-level accuracy of this file is limited (it contains placeholder-style content, e.g., referring to a `#define SINGLE_SCAN_SUPPORT 1` preprocessor macro that does not exist in the implementation — the actual mechanism is the `single_scan` field in `HUB75_I2S_CFG`). It serves as a starting point for user-facing documentation.

---

## Summary Table

| # | File | What Changed | Why |
|---|------|-------------|-----|
| 1.1 | `.h` | `MATRIX_ROWS_IN_PARALLEL` macro commented out; replaced by `uint8_t matrix_rows_in_parallel` instance variable | Enables runtime selection of 1 or 2 parallel rows |
| 1.2 | `.h` | `bool single_scan = false` added to `HUB75_I2S_CFG` | New mode flag; backward-compatible default |
| 1.3 | `.h` | `_single_scan` parameter added to `HUB75_I2S_CFG` constructor; `line_decoder` also now initialized | Allows configuration at construction; fixes pre-existing initializer omission |
| 1.4 | `.h` | Both `MatrixPanel_I2S_DMA` constructors initialize RGB bitmask members | Prevents undefined values before first pixel write |
| 1.5 | `.h` | `begin()` validates R2/G2/B2 and E pin for single-scan | Catches hardware wiring errors early |
| 1.6 | `.h` | `begin()` calls `initBitmasks()` after DMA alloc | Ensures masks are set at correct lifecycle point |
| 1.7 | `.h` | `setCfg()` sets `ROWS_PER_FRAME` and `matrix_rows_in_parallel` based on `single_scan` | Core value that determines DMA buffer depth |
| 1.8 | `.h` | In-class initializers for `PIXELS_PER_ROW`, `ROWS_PER_FRAME`, `MASK_OFFSET` removed | Old initializers ran before `setCfg()` and used wrong values |
| 1.9 | `.h` | Three `rgb*_clear_mask` private members + `initBitmasks()` added | Enables future mode-specific mask adjustment |
| 2.1 | `.cpp` | `setupDMA()` sets `ROWS_PER_FRAME` / `matrix_rows_in_parallel` before buffer alloc | DMA alloc size must reflect correct row count |
| 2.2 | `.cpp` | `updateMatrixDMABuffer(x,y,r,g,b)` uses RGB1-only + A-E addressing for single-scan | Single-scan panels address rows 0-31 directly via one RGB channel |
| 2.3 | `.cpp` | `updateMatrixDMABuffer(r,g,b)` skips RGB2 duplication in single-scan | Avoids corrupting address/OE bits in the same 16-bit word |
| 2.4 | `.cpp` | `clearFrameBuffer()` uses simplified address-bit write in single-scan mode | No driver-chip remapping needed; direct 1:1 row-to-address mapping |
| 3.1 | `.hpp` | `SINGLE_SCAN_32PX_HIGH` added to `PANEL_SCAN_TYPE` enum | New scan-type token for template specialization |
| 3.2 | `.hpp` | `kScanType` static constexpr added to `ScanTypeMapping<>` | Enables compile-time inspection of scan type in `if constexpr` branches |
| 3.3 | `.hpp` | `apply()` handles `SINGLE_SCAN_32PX_HIGH` — direct y % 32 passthrough | No coordinate remapping needed for linearly-scanned panels |
| 3.4 | `.hpp` | `drawDisplayTest()` uses forward panel numbering for single-scan | Matches physical layout intuition |
| 3.5 | `.hpp` | `setDisplay()` validates panel height, `single_scan` flag, and E pin at runtime | Cross-checks template config vs hardware config |
| 4   | `.md` | `SINGLE_SCAN_SUPPORT_CHANGES.md` created | High-level user-facing documentation |

---

*Report generated 2026-04-25. Based on diff of `origin/master...origin/feat/single-scan-support` in CosmicBitflip/ESP32-HUB75-MatrixPanel-DMA.*
