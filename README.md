# Brother QL CUPS Driver — Native ARM64

A native Apple Silicon (arm64) CUPS driver for Brother QL-series label printers.
No Rosetta 2 required.

> **Disclaimer:** This is an unofficial, independent project. It is not
> affiliated with, endorsed by, or supported by Brother Industries, Ltd.
> "Brother", "QL-800", "QL-1100" and "DK" are trademarks of their respective
> owners and are used here only to describe compatibility. Use at your own
> risk — see [LICENSE](LICENSE).

## Supported printers

| Printer  | Max width | Status |
|----------|-----------|--------|
| QL-1100  | 102mm     | Tested with DK-11241 (102×152mm die-cut) |
| QL-800   | 62mm      | Untested on hardware — reports welcome   |

Other label sizes are implemented from Brother's published geometry but have
not all been printed on real hardware. Please open an issue with what works
(or doesn't) for you.

## Supported media (QL-1100)

| PPD code  | Label              | Size          |
|-----------|--------------------|---------------|
| DK11241   | DK-11241           | 102mm × 152mm die-cut |
| DK11209   | DK-11209           | 62mm × 29mm die-cut   |
| DK11202   | DK-11202           | 62mm × 100mm die-cut  |
| DK22205   | DK-22205           | 62mm continuous       |
| DK22246   | DK-22246           | 103.6mm continuous    |

## Supported media (QL-800)

| PPD code  | Label              | Size          |
|-----------|--------------------|---------------|
| DK22206   | DK-22206           | 62mm continuous white |
| DK22205   | DK-22205           | 62mm continuous       |
| DK11209   | DK-11209           | 62mm × 29mm die-cut   |
| DK11202   | DK-11202           | 62mm × 100mm die-cut  |

For a 62mm continuous roll (DK-22205 / DK-22206) the QL-800 PPD also offers
fixed-length sizes that are printed as continuous tape and cut after the
label, matching the sizes of Brother's original Intel driver:

| PPD code      | Label                         | Size           |
|---------------|-------------------------------|----------------|
| C62x29        | 62mm Continuous, 29mm cut     | 62mm × 29mm    |
| C62x100       | 62mm Continuous, 100mm cut    | 62mm × 100mm   |
| CSmallAddress | Small Address Label           | 62mm × 29mm    |
| CShipping     | Shipping Label                | 62mm × 100mm   |
| CNameBadge    | Name Badge Label              | 62mm × 100mm   |
| CBinder5cm    | Binder 5 cm Spine             | 62mm × 209mm   |
| CPostage184   | Postage Label                 | 62mm × 184mm   |
| CPostage209   | Postage Label                 | 62mm × 209mm   |

Custom paper sizes with a width of 62mm (any length up to 1m) are printed
the same way. Only the `DK…` sizes are sent to the printer as die-cut labels.

## Requirements

- macOS on Apple Silicon (M1 or later)
- Xcode Command Line Tools: `xcode-select --install`
- CUPS (included with macOS)

## Install from a release (easiest)

1. Download `QL-Driver-ARM64-<version>.pkg` from the
   [Releases page](https://github.com/chickey/Brother-QL-macOS-arm64/releases).
2. Double-click it. The package isn't signed with an Apple Developer ID, so
   macOS will say it can't verify the developer. Open
   **System Settings → Privacy & Security**, scroll down and click
   **Open Anyway**, then run the installer again. Alternatively, from Terminal:
   ```bash
   sudo installer -pkg ~/Downloads/QL-Driver-ARM64-1.0.pkg -target /
   ```
3. Add the printer as described below.

The installer only runs on Apple Silicon Macs with macOS 11 or later; on an
Intel Mac it stops with a message.

## Build and install from source

```bash
git clone https://github.com/chickey/Brother-QL-macOS-arm64.git
cd Brother-QL-macOS-arm64

# Build the arm64 filter binary
make

# Install (requires sudo — writes to /usr/libexec/cups/filter)
sudo make install
```

## Add printer in macOS

1. Open **System Settings → Printers & Scanners**
2. Click **+** to add a printer
3. Connect the QL-1100 or QL-800 via USB
4. If macOS auto-selects Brother's Intel driver, click **Use** → **Select Software**
5. Search for **QL-1100** or **QL-800** — select the **(ARM64)** entry
6. Click **Add**

### Updating an existing printer

macOS keeps its own copy of the PPD for each printer, so after `sudo make install`
refresh it (replace the printer name with yours — `lpstat -p` lists them):

```bash
sudo lpadmin -p Brother_QL_1100_arm64 -P ppd/QL-1100.ppd
```

The "Printer drivers are deprecated" warning is harmless. Quit and reopen the
app you print from so its print dialog picks up the new options.

## Print options

These appear in the print dialog (under the printer's options) or can be
passed to `lp` with `-o Name=Value`.

| Option | Values | Default | Notes |
|--------|--------|---------|-------|
| Label Type (`MediaType`) | see tables above | DK11241 / DK22206 | The paper size chosen in the dialog takes priority |
| Auto Cut (`AutoCut`) | `True` (every label and at end), `EndOnly`, `False` | `True` | |
| Horizontal Adjust (`HorizontalAdjust`) | `Minus3` … `None` … `Plus3` (mm) | `None` | Positive moves the print right |
| Label Scale (`LabelScale`) | `100`, `98`, `96`, `95`, `92`, `90`, `Fit` | `98` | See below — leave the app's own Scale at 100% |
| Print Quality (`PrintQuality`) | `Quality`, `Speed` | `Quality` | Sets the printer's quality-priority flag |
| Halftone (`Halftone`) | `Binary` (logos/text), `Dither` (text/graphics), `ErrorDiffusion` (images) | `ErrorDiffusion` | How grey shading becomes dots |
| Brightness / Contrast | `-20` … `20` in steps of 5 | `0` | White stays white and black stays black |

Example:

```bash
lp -d Brother_QL_1100_arm64 -o PageSize=DK11241 -o LabelScale=Fit label.pdf
```

### Printable area and Label Scale

QL printers cannot print right to the edge of a die-cut label. On DK-11241
roughly 6mm at the top and bottom and 1.5mm at each side are unprintable
(the same limits Brother's own driver uses). The driver receives the full
label and shrinks it by **Label Scale** before trimming to the printable area,
so content near the edges survives. `Fit` shrinks just enough that nothing is
lost.

## Build a .pkg installer (optional)

The package is restricted to Apple Silicon by `pkg/distribution.xml`.

```bash
make pkg
# Output: build/QL-Driver-ARM64-1.0.pkg
sudo installer -pkg build/QL-Driver-ARM64-1.0.pkg -target /
```

## How it works

macOS printing uses CUPS. When you print any document:

```
App → PDF → CUPS → cgpdftoraster (converts to bitmap) → rastertoql (our filter) → USB → printer
```

`cgpdftoraster` renders each page as an 8-bit greyscale bitmap at 300dpi.
`rastertoql` scales it, applies brightness/contrast, halftones it to 1-bit,
and converts it to the Brother QL raster command format, writing to stdout
which CUPS pipes to the printer over USB.

The PPD files tell CUPS:
- What label sizes are available
- What resolution and colour depth to render at (300dpi, 8-bit grey)
- Which filter to use (`rastertoql`)
- Which options to show in the print dialog

## Troubleshooting

**Printer flashes red and doesn't print:** the printer rejected the job —
usually the label type selected doesn't match the roll loaded. Check the
paper size in the print dialog, then power-cycle the printer.

**Filter not found:** Make sure `/usr/libexec/cups/filter/rastertoql` exists and is executable (`chmod 755`).

**Check it's really arm64:**
```bash
file /usr/libexec/cups/filter/rastertoql
# Should say: Mach-O 64-bit executable arm64
```

**CUPS log** (enable detail with `cupsctl --debug-logging`):
```bash
tail -f /var/log/cups/error_log
```

**Test print:**
```bash
lp -d Brother_QL_1100_arm64 -o PageSize=DK11241 /path/to/test.pdf
```

## Adding more label sizes

1. Add a row to `QL1100_MEDIA[]` or `QL800_MEDIA[]` in `src/ql_protocol.h`:
   the width/length in mm the printer expects, and the printable dots.
2. Add matching `*MediaType`, `*PaperDimension`, `*ImageableArea` (the full
   page), `*PageSize` and `*PageRegion` entries to the relevant PPD file.
3. Rebuild with `make`, reinstall, and refresh the printer as above.

## Acknowledgements

Raster protocol details and print-head offsets were cross-checked against the
open-source [brother_ql](https://github.com/pklaus/brother_ql) project.

## Licence

MIT — see [LICENSE](LICENSE).
