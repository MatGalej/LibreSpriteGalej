# LibreSprite: Subsystems and Class Project Ideas

This guide describes the local source checkout inspected on November 12, 2026. LibreSprite is a desktop pixel-art editor and animation program, written primarily in C++. The goal for this class project is a small, useful feature whose behavior and implementation can be explained and demonstrated independently.

## Architecture at a glance

The application layer connects reusable libraries for document data, rendering, image effects, widgets, and operating-system access. The existing [source overview](src/README.md) describes these layers, although some build requirements and implementation details have changed since that overview was written.

```text
Application startup (src/main)
    |
Application coordination (src/app)
    |-- Menus, commands, dialogs, editor, timeline
    |-- Document editing and undo transactions
    |-- Drawing tools, file formats, preferences, scripting
    |
Reusable libraries
    |-- doc: sprites, images, layers, cels, frames, palettes, selections
    |-- render: compositing, zoom, onion skinning
    |-- filters: image effects
    |-- ui: widgets and events
    |-- she: operating-system and hardware abstraction
    |-- base, gfx, cfg, undo, clip, and other supporting libraries
```

## Important subsystems

### 1. Startup and application coordination

[src/main/main.cpp](src/main/main.cpp) contains `app_main()`, which initializes options, creates the platform system, and starts the application. [src/app/app.cpp](src/app/app.cpp) coordinates application initialization and execution. The application context tracks the active editing state; see [src/app/context.h](src/app/context.h) and [src/app/ui_context.cpp](src/app/ui_context.cpp).

### 2. Document and sprite data model

[src/doc/](src/doc/) represents the artwork itself: images, sprites, layers, cels, palettes, frame timing, and selection masks. A sprite contains animation frames and a layer hierarchy. A cel associates image content and placement with a particular layer and frame. Start with [sprite.h](src/doc/sprite.h), [layer.h](src/doc/layer.h), [cel.h](src/doc/cel.h), and [image.h](src/doc/image.h). [src/app/document.cpp](src/app/document.cpp) adds application-specific document behavior around the underlying model.

### 3. User commands, editing operations, and undo/redo

[src/app/commands/](src/app/commands/) contains user-facing actions such as changing canvas size, opening properties, and reversing frames. Commands define when an action is enabled and what it does. Registration uses [commands_list.h](src/app/commands/commands_list.h) and [command_factory.h](src/app/commands/command_factory.h).

[src/app/cmd/](src/app/cmd/) contains lower-level undoable edits. [document_api.cpp](src/app/document_api.cpp) provides editing operations, and [transaction.cpp](src/app/transaction.cpp) groups changes into undo history entries using [src/undo/](src/undo/). A document-changing feature should follow these existing patterns so undo, redo, and saved-state tracking work correctly. A display-only feature can often read the model without adding an undo operation.

### 4. Drawing tools and canvas interaction

[src/app/tools/](src/app/tools/) implements drawing behavior through strokes, controllers, point shapes, inks, and tool loops. [tool_loop_manager.cpp](src/app/tools/tool_loop_manager.cpp) is a useful entry point. [src/app/ui/editor/](src/app/ui/editor/) handles canvas interaction through states for drawing, moving pixels, scrolling, zooming, and playback. Changes here can affect many tools, so even a short change needs careful regression checks.

### 5. Rendering and animation playback

[src/render/](src/render/) composites document content into a displayed image and implements rendering support such as onion skinning and zoom. [render.cpp](src/render/render.cpp) is the central implementation. Rendering displays artwork; drawing tools modify artwork. Animation playback is coordinated by [play_state.cpp](src/app/ui/editor/play_state.cpp), with navigation controls in [ani_controls.cpp](src/app/ui/ani_controls.cpp) and frame/layer interaction in [timeline.cpp](src/app/ui/timeline.cpp).

### 6. Interface widgets, dialogs, menus, and themes

[src/ui/](src/ui/) supplies reusable controls and event handling. [src/app/ui/](src/app/ui/) builds editor-specific components such as the timeline, color bar, status bar, and workspace. Dialog layouts live in [data/widgets/](data/widgets/), menu and shortcut definitions in [data/gui.xml](data/gui.xml), and visual assets in [data/skins/](data/skins/). Language resources live in [data/languages/](data/languages/).

The build runs [src/gen/](src/gen/) to generate C++ widget wrappers from XML layouts. Edit the source XML and its application handler; generated `.xml.h` files in the build directory will be regenerated.

### 7. Color, palettes, and image effects

[src/doc/palette.h](src/doc/palette.h) represents palette data. [src/app/color.cpp](src/app/color.cpp) and the color controls in [src/app/ui/](src/app/ui/) connect color representations to the interface. [src/filters/](src/filters/) implements effects such as inversion, color replacement, convolution, and median filtering. [src/app/commands/filters/](src/app/commands/filters/) integrates those effects with dialogs, previews, and editing operations. Pixel-changing features must account for RGB, grayscale, and indexed images.

### 8. File loading, saving, and export

[src/app/file/](src/app/file/) contains format implementations, including ASE, PNG, GIF, JPEG, and QOI. [file_formats_manager.cpp](src/app/file/file_formats_manager.cpp) organizes available formats. [document_exporter.cpp](src/app/document_exporter.cpp) and the sprite-sheet commands handle export workflows. New formats and major export changes are relatively broad projects because formats differ in their support for layers, animation, transparency, and metadata.

### 9. Preferences, recent files, and recovery

[data/pref.xml](data/pref.xml) defines preference types; [src/app/pref/preferences.cpp](src/app/pref/preferences.cpp) loads and saves them. [src/cfg/](src/cfg/) and [src/app/ini_file.cpp](src/app/ini_file.cpp) support configuration storage. [recent_files.cpp](src/app/recent_files.cpp) maintains recent file/path lists, with menu presentation in [recent_files_menu.cpp](src/app/recent_files_menu.cpp). [src/app/crash/](src/app/crash/) contains document backup and recovery support.

### 10. Scripting and extensions

[src/app/script/](src/app/script/) integrates scripting with the editor, including script menus, permissions, and API bindings under `api/`. Supporting interfaces are in [src/script/](src/script/); [third_party/](third_party/) includes QuickJS and Delta. Bundled scripts live in [data/scripts/](data/scripts/). A script may be useful for prototyping an idea, but confirm whether the class requires modifying the compiled program itself.

### 11. Platform support and shared utilities

[src/she/](src/she/) abstracts platform services such as displays, input, fonts, and surfaces. This checkout enables the SDL2 backend by default and also contains an SDL3 build option. [src/base/](src/base/) supplies common utilities; [src/gfx/](src/gfx/) supplies geometry and graphics primitives; [src/clip/](src/clip/) handles clipboard support. Platform-level changes are usually a poor starting point for a small class feature.

### 12. Build system and verification

The root [CMakeLists.txt](CMakeLists.txt) configures dependencies and compilation. [src/CMakeLists.txt](src/CMakeLists.txt) builds the libraries and `libresprite` executable and copies runtime assets; [src/app/CMakeLists.txt](src/app/CMakeLists.txt) lists application sources and generates UI/preference code. Existing tests appear alongside source files as `*_tests.cpp`; the root `ENABLE_TESTS` option is off by default. Use the existing test infrastructure where appropriate, plus a manual demonstration of the feature.

## Example: how an interface feature reaches the data

Sprite Properties provides a manageable example to study:

1. The menu definition in [data/gui.xml](data/gui.xml) invokes the `SpriteProperties` command.
2. [cmd_sprite_properties.cpp](src/app/commands/cmd_sprite_properties.cpp) reads the active document and sprite through `ContextReader`.
3. [sprite_properties.xml](data/widgets/sprite_properties.xml) defines the dialog layout, which the build converts into a C++ wrapper.
4. The command fills the widgets from model properties such as size and frame count.
5. If the user changes the indexed transparent color, the command applies that edit through a transaction and `DocumentApi`.

This is a useful starting point because it demonstrates both reading existing data and making an undoable change.

## Potential class features

These are proposals based on the inspected local implementations, not confirmed upstream feature requests. The absence noted below is specific to the named interface or code path. Upstream issue and pull-request duplication has not been established.

| Feature | User benefit | Relative scope | Main starting points |
| --- | --- | --- | --- |
| Total animation duration in Sprite Properties | See how long an animation lasts without adding frame durations manually | Small; recommended | `cmd_sprite_properties.cpp`, `sprite_properties.xml`, `doc/sprite.h` |
| FPS entry in Frame Properties | Set familiar rates such as 12 FPS without calculating milliseconds | Small to medium | `cmd_frame_properties.cpp`, `frame_properties.xml` |
| Clear recent file history | Remove clutter or filenames before a classroom demonstration | Medium | `recent_files.*`, `recent_files_menu.cpp`, `ui/recent_listbox.cpp`, command registration |
| Jump to animation time | Navigate a long animation by elapsed milliseconds instead of frame number | Medium | `cmd_goto_frame.cpp`, `goto_frame.xml`, `doc/sprite.h` |

### Recommended: total animation duration

The inspected Sprite Properties dialog shows filename, color type, size, frame count, and transparent color, but no total duration. Add a read-only field such as `Animation duration: 450 ms (0.450 s)`.

Define this value as the sum of each frame's stored duration for one forward pass through the entire sprite. It should use individual frame durations because they can differ. This definition excludes playback speed overrides, tag-specific playback, and ping-pong repetition. The existing `Sprite::frameDuration()` and `totalFrames()` provide the needed information.

The likely core change is confined to the existing command and XML layout, with language resources handled according to the project's translation pattern. It does not require a new file format, preference, or editing operation. Use a sufficiently wide total for long animations.

Demonstration: create three frames lasting 100, 150, and 200 ms; verify that the dialog shows 450 ms. Change a frame duration and reopen the dialog; verify that the total updates. Also check a single-frame sprite and confirm that opening the dialog leaves the document unchanged.

### Alternative: FPS entry in Frame Properties

The inspected dialog accepts duration in milliseconds. Add an FPS entry that converts to the existing millisecond representation while preserving the current single-frame and frame-range behavior. For example, 10 FPS corresponds to 100 ms per frame.

The main design choices are rounding for rates such as 24 FPS, validation of zero or invalid input, and how the two fields stay synchronized. Applying the result should reuse the existing duration transaction. Demonstrate conversion, cancel behavior, selected-frame scope, undo, and redo. This is more interactive than a duration label, but has more edge cases.

### Alternative: clear recent file history

The inspected recent-files menu lists filenames or a disabled `Nothing` item; it has no clear-history action. Add an action that clears the recent file list and refreshes the menu and home view. Decide explicitly whether recent folder history is also cleared.

Persistence makes this larger than simply emptying a list: the current saving code writes populated entries, so obsolete configuration keys must also be removed to prevent old filenames from returning after restart. Demonstrate clearing, restarting, and opening a new file afterward. The action removes history entries only; it must not delete artwork files.

### Alternative: jump to animation time

The inspected Go to Frame dialog accepts a frame number. Extend navigation with an elapsed-time input, using cumulative stored frame durations to locate the frame containing that time. For frames lasting 100, 150, and 200 ms, time 120 ms should select the second frame.

Define boundary behavior, invalid input, and times beyond the animation length before implementation. This offers a clear algorithm to explain in class, but involves more validation and interface design than the recommended duration display.

## Local build and resource notes

The assignment also asks about build feasibility. These are local observations, not published minimum requirements:

- This checkout occupies approximately **905 MiB**, including existing build output and other local files. Its `build/` directory occupies approximately **843 MiB**; `src/`, `third_party/`, and `data/` occupy about **7.9 MiB**, **10 MiB**, and **23 MiB**, respectively. These figures do not include all externally installed dependencies or toolchains.
- The filesystem reported approximately **671 GiB available** during inspection.
- `build/bin/libresprite` exists and is approximately **10.8 MB**. Its existence is not verification that the current source builds or that the application runs correctly; neither was tested for this documentation task.
- Installed RAM could not be read under the current tool permissions. Peak build memory and application memory have not been measured. Record them with Activity Monitor during a baseline build and launch if the class requires measured resource use; they vary with build parallelism and document size.
- The actual root build file requires **CMake 4.1 or newer** and **C++20**. Some earlier requirements in [INSTALL.md](INSTALL.md) are outdated relative to this checkout. Use its macOS dependency list as a starting point and check configuration output against the actual build files.

Before changing a feature, establish a successful baseline build and launch using the existing setup. The existing `build/` cache is configured as `RelWithDebInfo`; `cmake-build-debug/` is configured as `Debug`. Keep the feature and its verification small enough to explain every changed line.

## Contribution context

Read [CONTRIBUTING.md](CONTRIBUTING.md) and [AGENTS.md](AGENTS.md) before implementation. This document proposes class-project work; it does not establish upstream interest or approval. The repository emphasizes contributor understanding and minimal scope. If you later intend an upstream contribution, check existing issues and pull requests and discuss the need with maintainers first, following the repository's restrictions on AI assistance and submissions.
