# Code structure

This is a map of the MuseScore Studio repository for someone opening it for the first time. It describes where the code lives and how the main pieces talk to each other. Coding style is in [CodeGuidelines.md](CodeGuidelines.md).  
   
MuseScore Studio is a Qt / QML application written in C++20. The program is a set of modules. Each module publishes interfaces and receives other modules' interfaces through a dependency-injection container. The score itself — notes, layout, and the `.mscz` file format — lives in `src/engraving`. Windows, toolbars, and dialogs live in QML next to the module that owns them. Shared infrastructure (the UI toolkit, audio engine, file system, logging) lives in the `muse` submodule and is not part of this repository's history.

```
QML windows and panels
        |
        v
Studio modules          src/          notation, project, playback, import/export, ...
        |
        v
Engraving engine        src/engraving score data, editing, layout, file format
        |
        v
Muse framework          muse/         IoC, UI toolkit, audio, drawing, actions
```

A user action travels downward. Opening a file starts in `project`, builds a score inside `engraving`, and wraps it in a `notation` object that the canvas can draw. Typing a note starts in QML, passes through `notation`, and changes the score in `engraving`. Layout then positions the symbols, and the canvas repaints.

## Repository root

Clone with submodules (`git clone --recurse-submodules`). Three directories are separate git repositories: `muse` (framework), `muse_deps` (third-party dependency recipes and prebuilt archives), and `dev-docs` (this documentation).

| Path | What it is |
| --- | --- |
| `src/` | MuseScore Studio modules. Almost all product code is here. |
| `muse/` | The [muse_framework](https://github.com/musescore/muse_framework) submodule. Shared modules used by Studio. |
| `muse_deps/` | Pins and recipes for libraries the build needs (audio, image, compression, and so on). |
| `share/` | Files installed with the application: instruments, templates, styles, icons, translations, soundfonts, plugins, workspaces. |
| `fonts/` | Text fonts and music fonts (Bravura, Leland, Edwin, and others), including SMuFL metadata. |
| `vtest/` | Visual tests. They render scores and compare the drawing against reference data. |
| `buildscripts/` | CMake helpers, CI scripts, and packaging. |
| `build.cmake` | The command-line build entry point (`cmake -P build.cmake`). |
| `tools/` | Developer utilities: code formatting, font tools, translations, MIDI helpers. |
| `hooks/` | Git hooks. `hooks/install.sh` installs a pre-commit formatter (Uncrustify). |
| `sandbox/` | Small programs that build one part of the code on its own (engraving, a web experiment). |
| `demos/` | Example scores. |
| `docs/` | Plugin API pages and older notes. Some of this material is outdated. |
| `dev-docs/` | Current developer documentation. |
| `test/` | An older test suite, kept around but not the place to add tests. |
| `.github/` | Issue templates and CI workflows. |

`build/` (if present) is a local build tree. Source does not live there.

Build configuration is selected with `MUSESCORE_BUILD_CONFIGURATION` in the top-level `CMakeLists.txt`:

| Value | Purpose |
| --- | --- |
| `app` | Desktop application. |
| `app-portable` | Windows portable build. |
| `app-web` | Experimental web build. |
| `vtest` | Visual tests, used in CI. |
| `utest` | Unit tests, used in CI. |

`MUSE_APP_BUILD_MODE` is `dev`, `testing`, or `release`.

## How the application is assembled

`src/app/main.cpp` starts the process. `src/app/appfactory.cpp` creates either a GUI application or a console application and calls `addModule(...)` for every module that this build includes. The GUI path is `MuseScoreGuiApp` in `src/app/internal/guiapp.cpp`. The console path is used for batch conversion and other headless runs.

Each module has a class that implements `muse::modularity::IModuleSetup` (see `muse/framework/global/modularity/imodulesetup.h`). The application drives every module through the same sequence, implemented in `muse/framework/global/internal/baseapplication.cpp`:

1. `registerResources` — Qt resources.
2. `registerExports` — put this module's implementations into the injector.
3. `registerUiTypes` — QML types.
4. `resolveImports` — look up interfaces registered by other modules and connect to them (for example, an import format registers itself as a reader).
5. `onPreInit`, `onInit`, `onAllInited`, `onStartApp` — startup work, in that order.

There are two injector scopes:

- **Global.** One instance for the whole process. Configuration, the instrument list, and format registries live here. Code asks for them with `muse::GlobalInject<Interface>`.
- **Context.** One per application window. A window's notation, playback controller, and UI state live here. Modules that need this override `newContext()` and return an `IContextSetup`. Code asks for them with `muse::ContextInject<Interface>`.

`src/CMakeLists.txt` turns modules on and off with `MUE_BUILD_*_MODULE` options declared in the top-level `CMakeLists.txt`. When a module is off, CMake builds the matching directory under `src/stubs/` instead. A stub exports the same interfaces with empty behavior, so the rest of the program still links.

### What a module looks like

Studio modules use the namespace `mu::<module>`. Framework modules use `muse::<module>`. Import/export code uses `mu::iex::<format>`.

A typical module directory:

```
src/notation/
  notationmodule.h/.cpp    registration and startup
  inotation.h              public interface (the leading "i")
  notationtypes.h          types other modules are allowed to see
  internal/                implementations; other modules do not include these
  qml/MuseScore/Notation/  QML module, URI MuseScore.Notation
  tests/                   GoogleTest unit tests
```

Other modules depend on the headers at the module root. They do not include `internal/`. Implementations are registered in `registerExports()` and retrieved through the injector, as in `src/notation/notationmodule.cpp`.

QML that belongs to Studio is registered as `MuseScore.<Name>`. QML that belongs to the framework is registered as `Muse.<Name>`.

## Studio modules

These are the directories under `src/`. `src/CMakeLists.txt` adds them in dependency order.

| Module | Role |
| --- | --- |
| `engraving` | The score: data model, editing commands, layout, painting, and the MuseScore file format. Described in the next section. |
| `notation` | The session API around one open score: selection, note input, undo, parts, playback state, accessibility. `INotation` is the facade other modules use. |
| `notationscene` | The canvas and the controls drawn on top of it: `NotationView`, note-input bar, toolbars, element popups, the navigator, the timeline. Mouse and keyboard handling lives here and calls `INotation`. |
| `project` | The file the user thinks of as a score. Open, save, recent files, autosave, cloud metadata. `INotationProject` is the facade. Readers and writers for every file format register here. |
| `importexport` | One small module per foreign format. Each one registers a reader, a writer, or both with `project`. |
| `playback` | Transport, mixer, and loop for the open score. It asks `engraving/playback` for musical events and the framework audio engine to play them. |
| `palette` | The palette panel: cells that insert elements into the score. |
| `propertiespanel` | The properties panel for the current selection. |
| `instrumentsscene` | The dialogs for choosing and editing instruments. |
| `appshell` | The main window: home page, notation page, menus, toolbars, first-launch setup. The web build uses `src/web/appshell` instead. |
| `preferences` | The preferences dialog. |
| `print` | Printing and PDF output. |
| `braille` | Braille display and export. |
| `musesounds` | UI and services around the Muse Sounds sampler library. |
| `converter` | Headless conversion (the console application's "convert this file to that format" path). |
| `app` | `main()`, module wiring, command-line parsing, application icons and shortcuts. |
| `web` | Experimental web shell (`appshell`, `appjs`) and a separate audio-engine target. |
| `macos_integration` | macOS Quick Look preview extension. |
| `stubs` | Empty stand-ins compiled when the real module is disabled. |

`notation`, `notationscene`, and `project` are easy to confuse. They own different things:

- `engraving` owns the musical data and how it is laid out on the page.
- `notation` owns the editing session for that data. `MasterNotation` wraps the full score. An excerpt notation wraps one part.
- `notationscene` owns the pixels and the input events.
- `project` owns the path, the dirty flag, and load/save. `EngravingProject` (inside `engraving`) is the in-memory container that load/save fills: the master score plus the extra data stored in an `.mscz` (styles, sound settings, and similar). The comment at the top of `src/engraving/engravingproject.h` explains that split.

### Opening a file

1. `project` decides the format and asks the registered `INotationReader` to read the bytes.
2. For `.mscz` (a zip archive) and `.mscx` (the same XML, uncompressed), reading is done by `src/engraving/rw`. `MscReader` / `MscWriter` in `src/engraving/infrastructure` handle the archive.
3. The reader builds a `MasterScore` and an `EngravingProject`.
4. `notation` wraps that score in an `INotation`.
5. `notationscene` paints it.

### Entering a note

1. The note-input bar and `NotationView` in `notationscene` receive the key or the click.
2. They call `INotationNoteInput` / `INotationInteraction`.
3. Those call the editing commands in `src/engraving/editing` (`noteinput.cpp` and the `edit*.cpp` files).
4. The command changes the DOM and records a tick range that needs layout. Undo is a transaction (`src/engraving/editing/transaction`).
5. `Score::doLayoutRange` runs layout for that range, then the view repaints.

User-facing commands (the things that have shortcuts and menu items) are declared with the framework `rcommand` module. The notation command list is `src/notation/notationcommands.h`. `NotationActionController` connects those commands to notation operations. Default shortcuts ship in `src/app/configs/data/shortcuts.json`.

## The engraving module

`src/engraving` is the largest body of code and the one most score bugs end up in. It can be built without the Qt UI (`tools/check_build_without_qt` does this). Layout and file I/O do not go through QML.

### The score tree

`EngravingObject` is the base of anything that lives in a score. `EngravingItem` (`src/engraving/dom/engravingitem.h`) is the base of things that have a position and can be drawn.

Musical structure and visual structure are stored separately. The musical structure is what editing changes. The visual structure is what layout rebuilds.

Musical structure:

- `MasterScore` is the full score. Parts the user has generated are `Excerpt` scores linked back to it.
- A `Score` owns `Part`s (instruments) and `Staff`s, and a list of measures.
- A `Measure` owns `Segment`s. A segment is one column at one tick: a clef, a key signature, a time signature, a chord or rest, a barline. `SegmentType` fixes the order of those columns.
- A `Chord` owns `Note`s. A rest is a `Rest`. Both are `ChordRest`.
- A `Spanner` (slur, tie, hairpin, volta, ottava, ...) is anchored to a start and an end and is tracked by the score, because it crosses measures.

Visual structure, produced by layout:

- `Page`s contain `System`s.
- A system places measures next to each other. A measure is owned by the score's measure list. Its layout parent is the system it was placed on. That distinction is documented on `EngravingItem::layoutParent()`.

### Subdirectories

| Path | What it is |
| --- | --- |
| `dom/` | Score objects. One class per symbol: `note`, `chord`, `measure`, `staff`, `slur`, `hairpin`, and the rest. `score.cpp` is the hub. |
| `editing/` | Operations that change the DOM: note input, paste, transpose, add and remove measures, and one `edit*.cpp` per element family. `transaction/` is the undo stack. |
| `rendering/score/` | Layout and painting of a full score. `ScoreLayout::layoutRange` is the entry point (`scorelayout.cpp`), called from `Score::doLayoutRange`. Per-symbol layout is the `*layout.cpp` files (`chordlayout`, `beamlayout`, `tupletlayout`, `pagelayout`, ...). `paint.cpp` draws the result. |
| `rendering/single/` | Draws one element by itself, for palettes and popups. |
| `rendering/editmode/` | Extra drawing while the user is dragging or editing. |
| `rw/` | Reading and writing `.mscx` / `.mscz`. The public classes sit in this directory (`mscsaver`, `mscloader`, `ireader`, `iwriter`). Versioned readers live in `read114`, `read206`, `read302`, `read400`, `read410`, `read460`, and `read500`. The current writer lives in `write/`. Read [rw/README.md](../src/engraving/rw/README.md) before changing a reader: several historical versions share code on purpose. |
| `style/` | Style defaults and text styles. The values the user edits in Format → Style are this data. |
| `playback/` | Turns a score into timed playback events (notes, dynamics, repeats). The audio engine that actually makes sound is in the framework. |
| `infrastructure/` | Archive I/O (`mscreader`, `mscwriter`), SMuFL font metrics, element ids, collision shapes, the skyline used by layout. |
| `api/v1/` | The plugin API exposed to QML plugins. |
| `types/` | Enums and small value types shared across the module (`Fraction` durations, element types, pitch). |
| `compat/` | Conversions that keep older scores opening correctly. |
| `data/` | Built-in engraving data (fonts metadata, defaults). |
| `accessibility/` | Accessibility descriptions of score elements. |
| `devtools/` | Debug overlays and inspection helpers, compiled when engraving devtools are enabled. |
| `qml/` | A small QML module (`MuseScore.Engraving`) for the few engraving types the UI needs to name. |
| `tests/` | Unit tests, plus `*_data/` directories of `.mscx` fixtures. |

Layout is incremental. An edit records the tick range it touched. `ScoreLayout::layoutRange` relayouts that range and rebuilds the systems and pages affected by it. Painting reads the geometry layout stored on each item.

## Import and export

`src/importexport` holds one module per format. Each module is optional in CMake. A few formats have a stub under `src/stubs/importexport` so a build without that format still provides the interface.

| Directory | Format |
| --- | --- |
| `musicxml` | MusicXML. |
| `midi` | Standard MIDI files. |
| `mei` | MEI. |
| `mnx` | MNX. |
| `musedata` | MuseData. |
| `guitarpro` | Guitar Pro. |
| `capella` | Capella, including an embedded RTF helper. |
| `bww` | Bagpipe Music Writer. |
| `bb` | Band-in-a-Box. |
| `ove` | Overture. |
| `tabledit` | TablEdit. |
| `audioexport` | Bounce the score to an audio file. |
| `imagesexport` | PNG, SVG, and similar page images. |
| `videoexport` | Video export. |
| `lyricsexport` | Lyrics text export. |
| `mlscore` | Optional, off by default. |

`.mscz` and `.mscx` are not in this list. They are the native format and are read and written by `engraving/rw`, with `notation` registering the writer.

## Playback and sound

Three layers, in three places:

- `src/engraving/playback` reads the score and produces the notes, dynamics, and repeats to play.
- `src/playback` is the product module: play button, seek, mixer, loop, and the QML for them.
- `muse/framework/audio` is the engine that mixes and outputs sound. `musesampler` and `vst` in the framework host MuseSampler and VST instruments. `src/musesounds` is the Studio UI on top of MuseSampler.

## User interface

The shell is `src/appshell/qml/MuseScore/AppShell`. `NotationPage.qml` is the score window. `HomePage/` is the start screen. Menus, the main toolbar, and first-launch dialogs are in the same QML module, with a C++ model next to each page (`notationpagemodel`, `appmenumodel`, ...).

Feature QML stays inside its module: the canvas in `notationscene`, palettes in `palette`, the properties panel in `propertiespanel`. Shared controls (buttons, lists, dialogs, docking) come from the framework modules `uicomponents`, `ui`, and `dockwindow`.

C++ models exposed to QML are ordinary `QObject`s registered by the module's `registerUiTypes()`.

## Muse framework

`muse/framework` is a separate repository, linked from the top-level `CMakeLists.txt` as `MUSE_FRAMEWORK_SRC_PATH`. `muse/framework/CMakeLists.txt` lists the modules. `global` is first and is linked into every other module; it holds the injector, async signals (`async/`), file and path helpers (`io/`), logging, and the application base class. Its own readme says to keep it small for that reason.

| Module | Role |
| --- | --- |
| `global` | Injector, application lifecycle, async, files, logging, containers. |
| `ui`, `uicomponents` | UI engine, navigation, and the shared QML control library. |
| `dockwindow` | Dockable panels. |
| `interactive` | Dialogs and popups opened from C++. |
| `actions`, `shortcuts_v2` | Action dispatch and keyboard shortcuts. The active shortcuts implementation is `shortcuts_v2`. |
| `rcommand`, `rcontrol` | Declarative commands and the remote-control channel used by UI and tests. |
| `draw` | Painting primitives shared by engraving and the UI. |
| `audio`, `audioplugins`, `musesampler`, `vst`, `mpe` | Audio engine, plugin hosting, and the musical performance events passed into it. |
| `midi`, `midiremote` | MIDI devices and MIDI remote control. |
| `accessibility` | Accessibility tree for the UI. |
| `cloud`, `network`, `update` | Accounts, HTTP, and the in-app updater. |
| `extensions` | Plugins and extensions. |
| `languages` | Translations. |
| `workspace` | Workspaces (which panels are open, and similar UI state). |
| `multiwindows` | More than one main window. |
| `learn`, `tours`, `toast` | Learn page, in-app tours, toast notifications. |
| `diagnostics` | Crash handling and diagnostics. |
| `automation`, `testflow` | Scripted UI automation used by tests. |
| `stubs` | Empty stand-ins for framework modules that are compiled out. |

## Resources the program loads

| Path | Contents |
| --- | --- |
| `share/instruments` | Instrument definitions. The source of truth is described in `share/instruments/README.md`. |
| `share/templates` | New-score templates. |
| `share/styles` | House styles, including the MSN style. |
| `share/locale` | Translations. |
| `share/icons` | Application icons. |
| `share/sound` | Built-in soundfonts. |
| `share/plugins` | Plugins shipped with the app. The API they call is `src/engraving/api/v1`. |
| `share/workspaces` | Default workspaces. |
| `share/wallpapers` | Score-view background images. |
| `share/extensions` | Extensions bundled with the app. |
| `fonts/` | Music and text fonts. SMuFL metadata is under `fonts/smufl`. |

## Tests

Add a test next to the code it covers, in that module's `tests/` directory. Engraving tests often pair a `.cpp` file with a `*_data/` directory of `.mscx` scores.

`vtest/` renders scores and compares the drawing with stored references, which is how layout regressions are caught. See `vtest/README.md`.

`share/testflowscripts` contains scripted UI flows. The `test/` directory at the repository root is the old suite.

Unit tests are part of the `utest` build configuration. Visual tests are part of the `vtest` configuration.

## Where to look

| You want to... | Start here |
| --- | --- |
| Change how a symbol is stored or behaves | `src/engraving/dom/<symbol>.cpp` |
| Change an editing operation (transpose, paste, note input, tuplets, ...) | `src/engraving/editing/` |
| Change spacing, beaming, or page layout | `src/engraving/rendering/score/` |
| Change how a `.mscz` is read or written | `src/engraving/rw/` and `rw/README.md` |
| Change selection, the cursor, or undo as the UI sees it | `src/notation/internal/` |
| Change the canvas, hit testing, or a popup on the score | `src/notationscene/` |
| Add a menu command or a shortcut | `src/notation/notationcommands.h`, then `shortcuts.json` |
| Change open, save, or recent files | `src/project/` |
| Add or fix an import format | `src/importexport/<format>/` |
| Change playback timing of a marking | `src/engraving/playback/` |
| Change the mixer or the play panel | `src/playback/` |
| Change the main window or the home page | `src/appshell/qml/MuseScore/AppShell/` |
| Change a shared button, dialog, or dock | `muse/framework/uicomponents` or `muse/framework/dockwindow` |
| Change sound output or instrument hosting | `muse/framework/audio`, `musesampler`, `vst` |
