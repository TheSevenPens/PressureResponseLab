# Architecture

`PressureResponseLab` is a small WPF desktop tool for characterizing the pressure curve of a stylus pen. It reads pen pressure from a digitizer (via WinTab) and a reference physical force from a USB scale (via a serial port), pairs them on demand, and plots/exports the resulting pressure-response curve.

## Solution layout

```
PressureResponseLab.slnx          # SLNX solution
├── SevenLib/                     # Class library — math, geometry, stylus types
├── PressureResponseLab/          # WPF WinExe — application
└── lib/PenSession.Managed/       # Vendored binary release of PenSession.Managed
```

## Projects

### `SevenLib` (class library, `net10.0-windows`)

Self-contained utility library. No project or NuGet dependencies. Used by `PressureResponseLab` for numerics and trig — nothing in this library knows about WPF, WinTab, or the scale.

| Namespace | What lives there |
|---|---|
| `SevenLib.Geometry` | `Point`, `PointD`, `Size`, `SizeD` value types |
| `SevenLib.Numerics` | `MovingAverage` (windowed mean), `IndexedQueue<T>`, `EMASmoother`, `EMAPositionSmoother`, `Interpolation`, `OrderedRange`/`OrderedRangeD`, `RangeD`, `SimpleCurve` |
| `SevenLib.Stylus` | `StylusButtonId`, `StylusButtonState`, `PointerData` |
| `SevenLib.Trigonometry` | `Angles`, `TiltAA` (azimuth/altitude), `TiltXY` |

Of these, only `Numerics.MovingAverage` and `Numerics.IndexedQueue<T>` are actively used by the application today; the rest is carry-over from the source repo and is preserved as-is in case future features need it.

### `PressureResponseLab` (WPF WinExe, `net10.0-windows`, `UseWPF=true`)

The application. Single-window UI; no MVVM. State is held in a single mutable `AppState` object that the window reads from and writes to directly.

External dependencies:

| Dependency | Source | Purpose |
|---|---|---|
| `PenSession.dll` | File reference: `..\lib\PenSession.Managed\PenSession.dll` | Stylus/digitizer abstraction; provides `IPenSession`, `PenPoint`, etc. Marked `Private=true` so it deploys next to the exe. |
| `OxyPlot.Wpf` | NuGet 2.1.2 | The pressure-response chart (`PlotView`). |
| `System.IO.Ports` | NuGet 10.0.3 | Serial port I/O for the USB scale. |
| `SevenLib` | Project reference | See above. |

Entry point: `App.xaml` `StartupUri="MainWindow.xaml"`. `Program.cs` is empty (just a comment).

## Component map

```
                       ┌──────────────────────────────┐
                       │         MainWindow           │
                       │  (UI + orchestration)        │
                       └────────────┬─────────────────┘
                                    │ owns
                                    ▼
                       ┌──────────────────────────────┐
                       │           AppState           │
                       │  (single mutable container)  │
                       └─┬───────────┬────────┬───────┘
                         │           │        │
                ┌────────▼──┐  ┌─────▼────┐  ┌▼─────────────────┐
                │ IPenSession│  │ScaleSess.│  │PressureRecordColl│
                │ (PenSession│  │+SerialPort│  │ + PressureRecord │
                │  .Managed) │  │(scale)    │  │  (logged points) │
                └───────────┘  └──────────┘  └──────────────────┘
```

### `MainWindow` (MainWindow.xaml + .xaml.cs)

The single window. Three logical regions in the XAML grid:

- **Left** — live pen telemetry (raw / normalized / smoothed pressure, azimuth, altitude, tilt X/Y, button state checkboxes) and the scale start/stop controls.
- **Center** — the OxyPlot pressure-response chart with an axis-range selector (`Default` / `Full` / `IAF` / `max`).
- **Right** — record list (DataGrid), record/clear/export buttons, and the metadata fields (brand, pen, family, date, user, tablet, driver, OS, tags, notes) that get bundled into the JSON export.

The window owns the lifecycle of both capture sessions and the polling timer.

### `AppState` (AppState.cs)

A plain mutable POCO that holds everything the window reads or writes. Not thread-safe — all field updates happen on the UI thread (the scale background task marshals back via `Dispatcher.Invoke`).

Notable members:
- `IPenSession? PenSession` — the live pen session.
- `ScaleSession? ScaleSession` — holds the moving-average smoother for logical pressure.
- `SerialPort? SerialPort`, `CancellationTokenSource? ScaleCts`, `bool ScaleIsReading` — scale plumbing.
- `double PhysicalPressure`, `double LogicalPressure` — last-seen samples used when the user clicks Record.
- `PressureRecordCollection? RecordCollection` — the logged data points.
- `IndexedQueue<double>? QueueLogical` — rolling buffer of smoothed pressure values (size `LogicalPressureQueueSize = 400`).

### Pen capture

Uses **PenSession.Managed** rather than raw WinTab. `MainWindow` constructs the session via `PenSessionFactory.Create(InputApi.WintabSystem)`, starts it on `Window.Loaded` with the HWND from `WindowInteropHelper`, and drives it with a polled loop:

- A `DispatcherTimer` (interval `PenPollIntervalMs = 8`) ticks on the UI thread.
- Each tick: if `IPenSession.HasNewData`, call `DrainPoints()` and feed each `PenPoint` to `ProcessPenPoint`.
- `ProcessPenPoint` reads `Pressure` (normalized against `IPenSession.MaxPressure`), `TiltX`/`TiltY`, `Azimuth`/`Altitude`, and `ButtonAction`/`ButtonNumber`, then updates the labels, `AppState.LogicalPressure`, the moving average, and `QueueLogical`.

Button mapping from `PenSession.PenButtonNumber` to UI checkboxes:

| PenSession | UI |
|---|---|
| `Tip` | `checkBox_tipdown` |
| `Barrel1` | `checkBox_lowerbuttondown` |
| `Barrel2` | `checkBox_upperbuttondown` |

Any button release also clears the moving average — preserved from the original app's "release resets the smoother so the next press starts clean" behavior.

Stop path (`StopPenSession`): stop and detach the timer, then call `IPenSession.Stop()`.

### Scale capture

Independent of the pen path. The user picks a COM port, hits **Start**, and `StartScaleSession` opens the port and runs `ReadSerialPortAsync` on a background task:

- Loop reads `SerialPort.ReadLine()` whenever `BytesToRead > 0`.
- `ParseScaleLine` strips trailing `M`/`g` suffix and parses a double.
- On success, `AppState.PhysicalPressure` is set and the force label is updated via `Dispatcher.Invoke`.

Stop path: cancel the `CancellationTokenSource` (and replace it for the next run), close+dispose the port.

### Records

- `PressureRecord` — immutable `(PhysicalPressure, LogicalPressure)` pair.
- `PressureRecordCollection` — wraps `List<PressureRecord>` with `Add` / `Clear` / `ClearLast` and JSON/text formatters (`GetRecordsJSON`, `GetRecordsText`).

A "record" is created when the user clicks **Record (Ctrl+R)** — the window snapshots the latest `AppState.PhysicalPressure` and the smoothed pressure from `ScaleSession.LogicalPressureMovingAverage.GetAverage()`, appends to the collection, and re-renders the DataGrid + chart.

### Plot

OxyPlot `PlotView` bound in XAML. `MainWindow.updatedata()` rebuilds the model on every change: a single `LineSeries` of all records plus a one-point `ScatterSeries` for the row currently selected in the DataGrid. `ApplyAxisRange` switches X/Y limits based on the four `comboBox_axisRange` modes.

### Persistence

- **Export** — writes `{ID}_{date}.json` to `MyDocuments`. JSON shape includes the metadata fields plus a `records: [[physical, logical%], ...]` array.
- **Drag-and-drop** — dropping a `.json` file on the window calls `LoadJSONFile`, which deserializes via `System.Text.Json` into `PressureTestData`, replaces the record collection, and rehydrates the metadata textboxes. The loaded path is remembered so **Save (Ctrl+S)** writes back to the same file.

## Threading model

| Code | Thread |
|---|---|
| WPF UI, `DispatcherTimer` ticks, `ProcessPenPoint`, all label/checkbox updates | UI thread |
| `IPenSession` internals (driver callbacks) | Owned by PenSession; samples are queued and drained from the UI thread |
| `ReadSerialPortAsync` loop | Background `Task` started from `button_start_Click`; marshals UI updates via `Dispatcher.Invoke` |

`AppState` has no synchronization. This is safe because all writes happen on the UI thread (scale parsing assigns `PhysicalPressure` from the background thread, but it's a single double write — torn reads aren't a concern in practice for this app's use).

## Lifecycle

1. **`MainWindow` ctor** — instantiates `IPenSession`, `ScaleSession`, `PressureRecordCollection`, `CancellationTokenSource`; populates COM-port combo; initializes the chart.
2. **`Window.Loaded`** — `StartPenSession`: gets HWND, calls `IPenSession.Start(hwnd)`, starts the `DispatcherTimer`.
3. **User actions** — record/clear/export/load, scale start/stop. Keyboard shortcuts handled in `Window_PreviewKeyDown` (Ctrl+R/L/C/A/S/T).
4. **`Window.Closing`** — `StopPenSession` (stop timer, stop session), `StopScaleSession` (cancel CTS), close+dispose serial port, dispose CTS.

## Why this layout

- **`SevenLib` separate from the app.** Keeps the math/types reusable and free of WPF/WinTab. The application would still build if `SevenLib` were swapped for a NuGet package later.
- **`lib/PenSession.Managed/` vendored.** PenSession.Managed is distributed as a release zip rather than a NuGet package, so the binaries are committed under `lib/` and pulled in via a `<Reference HintPath>`. This keeps the build hermetic — no separate download step — at the cost of versioning the DLLs in git.
- **Polled `IPenSession` instead of a callback API.** PenSession.Managed exposes `HasNewData` + `DrainPoints()`. A `DispatcherTimer` keeps everything single-threaded on the UI; the alternative (event-per-packet on a background thread plus `Dispatcher.Invoke`) was the V1 model and added ordering complexity for no real benefit at ~120 Hz.
- **Single mutable `AppState`.** With one window and no MVVM, a flat state bag is the simplest thing that works. All threading is funneled through the UI dispatcher so the absence of locks is intentional, not accidental.
