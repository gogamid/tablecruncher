# AGENTS.md — Tablecruncher

Tablecruncher is a desktop CSV editor written in **C++17** with **FLTK 1.4.x** for the UI
(C++17 / FLTK is the *only* external runtime dependency). It is a single CMake target —
there is no monorepo, no package manager, and **no test suite**.

## Build

FLTK is **not vendored**. You must obtain FLTK 1.4.x and build it yourself first
(see `BUILD.md`), then point CMake at it.

```bash
# macOS (FLTK built in-tree: the root must contain BOTH `FL/` headers and `lib/libfltk.a`)
mkdir -p build && cd build
cmake -DFLTKDIR=/path/to/fltk-1.4.3 -DMACOS_VERSION=10.15 ..
cmake --build . -j$(sysctl -n hw.ncpu)
```

Linux is the same minus `-DMACOS_VERSION`; add `-DAPPIMAGE=1` for an AppImage build.

Non-obvious build facts:

- **`-DMACOS_VERSION=<x.y>` is mandatory on macOS.** `CMakeLists.txt` unconditionally emits
  `-mmacosx-version-min=${MACOS_VERSION}`; if the variable is empty the compiler flag is
  malformed and the build fails with a confusing error. Use the *same* value you used to
  build FLTK.
- **The CMake binary dir must be named `build/` at the repo root.** `scripts/create-bundle.sh`
  hardcodes `./build/dist` and runs with `WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}`.
  Configuring into e.g. `out/` silently breaks the macOS bundle step.
- The macOS `.app` bundle is produced by a **POST_BUILD hook on every build**
  (`scripts/create-bundle.sh`), so `cmake --build .` already yields
  `build/dist/Tablecruncher.app`. The extra `create_bundle` target is a duplicate of it.
- Binary output lands in `build/dist/Tablecruncher` (`RUNTIME_OUTPUT_DIRECTORY`).
- **New `.cpp` files must be added to the explicit `SOURCES` list in `CMakeLists.txt`** —
  there is no globbing. Headers need no registration.
- `CMakePresets.json` only has a Windows preset with placeholder FLTK paths; it is not usable
  as-is on macOS/Linux.
- `cmake --build . --target clean` / `custom_clean` (the latter removes `build/dist`).
- Debug output in the code is gated on `#ifdef DEBUG`, which CMake does **not** define for
  `CMAKE_BUILD_TYPE=Debug`. Add `-DCMAKE_CXX_FLAGS=-DDEBUG` if you want those traces.

## Versioning (dual, easy to get wrong)

The user-facing version is derived from `project(... VERSION ...)` in `CMakeLists.txt`:

| CMake `VERSION` | Displayed `APP_VERSION` |
|---|---|
| `1.8.9`  | `1.8-beta9` (PATCH < 10 → beta) |
| `1.8.10` | `1.8` (PATCH == 10 → final) |
| `1.8.11` | `1.8.1` (PATCH > 10 → post-final, PATCH-10) |

`scripts/version.txt` holds the **same** user-facing string (`1.8.1`) and is what the
AppImage script and the Windows Inno Setup installer read. **Both places must be updated
together** or packaging will produce mismatched artifact names. `docs/dev/RELEASE.md`
describes the tag/publish flow (its version examples are older than the table above — trust
`CMakeLists.txt`).

## Architecture

MVC-ish, split across `src/` (all classes, one executable):

| File | Role |
|---|---|
| `main.cpp` | Defines the **global singletons**: `app` (`CsvApplication`), `windows[]` (`CsvWindow[10]`), `macro` (`Macro`), and the three `Fl_Preferences` objects `preferences`, `macros`, `file_preferences`. |
| `csvapplication.cpp/.hh` | The application singleton: menu callbacks, dialogs, CSV format guessing, encoding detection, update check. ~3700 lines, the largest file. |
| `csvwindow.cpp/.hh` | One document window (FLTK window + toolbar + status bar + menu bar). Owns the undo stack. |
| `csvtable.cpp/.hh` | Controller for the data of one window (the "Model" facade). |
| `csvdatastorage.cpp/.hh` | Actual storage. Each row is **one `std::string`**; fields inside it are separated by the byte `0xFA` (chosen because it is invalid UTF-8, so it cannot occur in cell data). |
| `csvgrid.cpp/.hh` | The `Fl_Table` view widget. |
| `csvmenu.cpp/.hh` | Builds the menu bar and dispatches clicks. |
| `csvparser.cpp/.hh` | FSM CSV parser (multi-encoding, BOM handling). |
| `csvundo.cpp/.hh` | Undo snapshot type. |
| `csvwidgets.cpp/.hh` | FLTK widget subclasses (`My_Fl_*`, themed toolbar buttons). |
| `colorthemes.cpp/.hh` | The four themes and color lookup by key. |
| `helper.cpp/.hh` | Static utilities: number parsing, encoding conversion, path helpers, font lookup. |
| `globals.hh` | All shared `#define` constants and the `CsvDefinition` struct (encoding, delimiter, quote style, BOM, linebreak). |

### Globals are `extern` in every translation unit

`app`, `windows[]`, `macro`, `preferences`, `macros`, `file_preferences` are defined **only**
in `main.cpp`. Any new `.cpp` that touches them must declare them itself, e.g.:

```cpp
extern CsvApplication app;
extern CsvWindow windows[];
```

### Macros (JavaScript via duktape)

`src/macro.cpp` embeds duktape and exposes `setCell`, `getString`, `getInt`, `getFloat`,
`flagRow`, `println` to user macros. `ROWMIN/COLMIN/ROWMAX/COLMAX/_ROWS/_COLS` are injected
as a prelude before the user's source.

Two things an agent will not infer from the docs:

- **The same duktape context is reused for every macro run and is never reset**, so top-level
  `var` declarations persist across executions. That is the "global variables in macros"
  feature from the 1.8 release notes — but it also means a broken macro can leave stale state
  behind.
- **The regex Find & Replace engine is implemented in JavaScript**, not in C++:
  `CsvTable::createJsFindSubstring()` and `CsvTable::replaceUtf8String()` build JS snippets
  and run them through `Macro::execute()`. Changes to the macro API therefore change
  search/replace behaviour.
- Known bug in `Macro::Macro()`: `__println` is registered pointing at
  `internalStoreStrings` (the `internalPrintln` handler is never registered). A macro calling
  `__println` will silently overwrite the stored string results instead of printing.

### Wiring up menus and UI

- Menu items are added in `CsvMenu::init()` with **path-style labels** (`"&File/&Open ..."`).
- Clicks are dispatched in `CsvMenu::MyMenuCallback()` by **string comparison of
  `item->label()`** — the label in `init()` and the label in the `strcmp` chain must match
  exactly (use the `TCRUNCHER_MENUTEXT_*` macros for shared labels).
- macOS uses `Fl_Sys_Menu_Bar` and puts Preferences/About in the native app menu
  (`Fl_Mac_App_Menu::custom_application_menu_items`); `&File/&Quit`, `&Edit/&Preferences...`
  and `&Help/...` are `#ifndef __APPLE__` only.
- Always call `Fl_Choice::add(label, shortcut, callback, data)` with **four arguments**.
  The single-argument overload splits the label at `|` and breaks custom-delimiter entries.
- New icons are XPM files in `src/icons/`, included into the `ui_icons` namespace in the
  header that uses them.
- `src/icons/abouticon_svg.hh` is a hand-copied copy of `assets/artwork/app-icon.svg` — update
  both together.

### Preferences

Stored via FLTK `Fl_Preferences` under the folder `com.tablecruncher`, in three files:
`tablecruncher` (app prefs), `macros` (user macro sources), `files` (per-file column widths,
header-row state, keyed by a path hash). Read with
`app.getPreference(&preferences, KEY, default)`, write with `preferences.set(...)`.
New keys go in `src/globals.hh` as `TCRUNCHER_PREF_*`.

## Verifying changes

There is **no test target, no CI, and no lint/format config**. The only test code in the repo
is `external/utf8-cpp-utils/test/` (doctest), which is not wired into CMake.

So verification is manual: build, launch `build/dist/Tablecruncher` (or the `.app`), and
exercise the affected menus. For data-layer changes, open a CSV and check the status bar /
Info window row and column counts. Macro changes can be exercised from
**Macro > Execute Macro ...**.

## Style

- Tabs for indentation; `if( cond )` with a space after the keyword; braces on the same line.
- Every source file starts with the GPL-3.0-or-later SPDX header block — copy it verbatim
  into new files.
- Doxygen comments on classes/methods (`/** \brief ... */`) are the norm; `src/Doxyfile`
  generates docs into `docs/doxygen/` (gitignored). Run `doxygen src/Doxyfile` from the repo
  root.
- External sources are vendored under `external/` (duktape, cpp-httplib, nlohmann/json,
  utfcpp) — do not modify them.
  `external/utf8-cpp-utils/utf8_cpp_tables.hh` is generated by
  `external/utf8-cpp-utils/create-unicode-tables.py` and is checked in.

## Packaging and release

- macOS signing/notarization: `scripts/sign-and-notarize.sh <version>` — it expects the
  `.app` in the current working directory and hardcodes the Apple Developer ID and team ID.
- Windows: `scripts/buildwin.bat` (Resource Hacker for the icon + Inno Setup using
  `scripts/version.txt`), `scripts/win-setup.iss` defines the installer and the `.csv`
  file association.
- Linux: `scripts/build_appimage.sh` reads `build/dist/Tablecruncher` and `scripts/version.txt`.
- `docs/user/docs-1.8.ascii` is the asciidoc user manual; `docs/user/release-notes.txt` is the
  changelog. Update both when user-visible behaviour changes.

## Constraints

- `CONTRIBUTING.md` states the project is **not accepting pull requests** yet (legal/licensing
  review pending). Do not open PRs; work in the local checkout and report issues.
- The app is GPL v3; the README asks forks to use a different name and logo.
- The app is feature-frozen-ish: "Windows builds are still experimental" per
  `CONTRIBUTING.md`.
