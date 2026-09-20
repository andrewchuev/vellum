# Vellum Reader

[![Download Latest Release](https://img.shields.io/github/v/release/andrewchuev/vellum?style=for-the-badge&label=Download&color=blue)](https://github.com/andrewchuev/vellum/releases/latest)

A minimalist, high-performance FB2 ebook reader optimized specifically for **E-Ink devices** (like Onyx Boox Page). Built from scratch with a focus on speed, high contrast, and battery efficiency.

## Screenshots

<p align="center">
  <img src="screenshots/library.png" width="30%" title="Library View" alt="Library View" />
  <img src="screenshots/empty_state.png" width="30%" title="Empty State View" alt="Empty State View" />
  <img src="screenshots/reader.png" width="30%" title="Reader View" alt="Reader View" />
</p>

<p align="center">
  <img src="screenshots/reader_menu.png" width="30%" title="Reader Menu" alt="Reader Menu" />
  <img src="screenshots/gestures_guide.png" width="30%" title="Gestures Guide Dialog" alt="Gestures Guide Dialog" />
</p>

* **Library View**: Shows cover images, titles, authors, read progress, and book series information. Built with Material Design Exposed Dropdown menus.
* **Empty State View**: Displays a clean book vector illustration when the library or search result list is empty.
* **Reader View**: Pure, distraction-free high-contrast reading text layout with clock and progress details in the bottom status bar.
* **Reader Menu**: Configurable font, layout, theme, and mode options.
* **Gestures Guide Dialog**: On-demand overlay detailing all tap, swipe, and touch shortcuts.

---

## Key Features

- **Optimized for E-Ink**: No animations, no gradients, pure black-on-white rendering.
- **Hardware Integration**: Custom support for Onyx Boox E-Ink screens (refresh modes, dual warm/cold brightness control).
- **Universal Compatibility**: Works on any Android 11+ device thanks to an abstraction layer over hardware-specific SDKs.
- **Fast FB2 Engine**: Custom XmlPullParser-based streaming parser handles large books with minimal memory footprint; titles, subtitles, epigraphs, quotations and poems are all rendered.
- **Support for .fb2 and .fb2.zip**: Open compressed books directly from your storage.
- **PDF Reader**: Fixed-layout PDFs open in their own reader, powered by Pdfium (so it works the same on every Android version, including E-Ink devices without Google services). One page at a time, fit-to-width by default with in-page scrolling on tall pages, pinch and double-tap zoom, pan, tap zones and volume keys for turning pages. It has a real table of contents read from the document outline, full-text search with all matches highlighted on the page, **long-press a word to select it and look it up in a dictionary** (same flow as in FB2), "go to page", password-protected documents, **night mode** and **margin cropping** (zooms to the content of each page, a big win on small screens). Reading progress is remembered; title, author and a first-page cover come from the document. Storage scanning imports `.pdf` files next to FB2 books, and the format is detected from the file content, not its name.
- **Instant Loading & Background Pagination**: The book opens instantly (< 2ms) by calculating and rendering the current page first. Full pagination is calculated asynchronously in the background on a thread pool (`Dispatchers.Default`), ensuring zero UI lag.
- **Justified Text Alignment**: Native full-width inter-word justification alignment (`Layout.JUSTIFICATION_MODE_INTER_WORD`) for clean and balanced layout on both edges.
- **Library Management**: Persistent local library with covers, metadata, and reading progress tracking. Includes options for physical file deletion when removing books.
- **Folder-Specific Scanning**: Multi-choice checklist dialog during storage scan to target specific folders (`Books`, `Download`, `Documents`) or scan the entire device.
- **Backup & Restore**: Export your per-book preferences and reading progress to a JSON file at a location of your choice (system file picker), and restore it later onto the books present in your library.
- **Automated Book Scanner**: Background storage scanner recursively searches external storage for `.fb2` and `.zip` files, extracts metadata, saves cover images, and registers books to the library, automatically skipping duplicates.
- **E-Ink Friendly Grouping & Filtering**: Group books by **Authors** and **Series/Sequences** using alphabetical navigation folders. Sort books in a series by sequence order, and filter library lists instantly by status (**Reading**, **Unread**, **Finished**).
- **Reactive UI**: Library updates automatically using Kotlin Coroutines Flow when books are added, opened, or scanned.
- **Per-Book Settings**: Remembers font size, line spacing, font family, and margins individually for every book. Font size and spacing changes render instantly on screen without blocking loading dialogs.
- **Navigation Options**:
    - **Physical Buttons**: Use volume keys to turn pages.
    - **Touch Zones**: Left/Right sides for paging, Center for menu.
- **Footnotes & Bookmarks**:
    - **Footnotes**: View annotations and author notes in a popup without leaving the current page.
    - **Bookmarks**: Save bookmarks at any position, listed in a modern dialog with dividers, padding, and styled high-contrast controls.
- **Dictionary Lookup Integration**: Long-press any word in the book to highlight it and look it up instantly using standard Android dictionary/translation intents (works with ColorDict, GoldenDict, etc.).
- **Custom Fonts Support**: Scan and load external `.ttf` or `.otf` fonts dynamically from `/sdcard/Fonts` directory.
- **Full-Text Search**: Search for words or phrases inside the currently open book, preview matching snippets, jump to the matches, and view highlighted occurrences on screen.
- **Advanced E-Ink Anti-Ghosting**: Full-screen updates are triggered after page turns and automatically on all dialog dismissals to instantly clear ghosting outlines and artifacts.
- **Night Mode**: Software-level color inversion for comfortable low-light reading.

## Engineering Notes

### Reliability
- **Thread-safe pagination**: `PaginationController` works on a private copy of the text paint and is otherwise stateless, so a background pass can no longer race with UI changes. The pass is cooperatively cancellable, and the table of contents is derived from the book itself (available immediately, no duplicates).
- **Layout matches rendering**: pagination honours the selected line spacing (it used to assume 1.2 always, which clipped text at larger spacings), and reading progress is not overwritten with a bogus 100% while pagination is still in progress.
- **No content loss**: subtitles, epigraphs, quotations and poem lines are now laid out and displayed; previously they were parsed and searchable, but never shown.
- **Durable covers**: covers live in app-private storage (`filesDir/covers`) instead of the cache dir the system may purge, are downsampled while decoding (bounded memory for huge images), and are removed together with their book.
- **Safe database evolution**: Room schemas are exported (`app/schemas`), and destructive fallback applies only to the pre-export legacy versions, so a future schema change without a `Migration` fails loudly instead of silently wiping progress and bookmarks.
- **Backup through the system file picker (SAF)**: no more writing into `/sdcard/Download` (which silently failed without all-files access). Unset per-book preferences stay unset after a restore, a zero margin survives a round trip, and one malformed entry no longer aborts the whole restore.
- **Lifecycle correctness**: rotating the screen no longer re-parses the book (the ViewModel re-lays it out for the new size), the opening/scan/backup dialogs are state-driven and never leak windows, the clock/battery ticker only runs while the screen is visible, and input streams are closed.
- **Responsive search**: book search runs off the main thread (results capped at 500); highlighting moved into `ReaderView`.

### Architecture
- **MVVM with state-driven UI**: `ReaderViewModel` and `LibraryViewModel` (`AndroidViewModel`s built with the `viewModelFactory { initializer { } }` DSL) expose `StateFlow` state plus one-off events. ViewModels no longer format user-facing strings: events carry string resource ids, and scan/backup outcomes are typed (`ScanState`, `BackupResult`).
- **Pure, tested list logic**: library filtering/grouping lives in `buildLibraryItems()`, a plain function covered by fast JVM tests.
- **Shared building blocks**: `CoverStorage`, `BitmapDecoder`, `FontChoice.toTypeface()` and `ReadingTheme.effective()` replace logic that used to be duplicated across the scanner, reader and views.
- **Single composition root**: `AppContainer` (held by `VellumApplication`) wires everything; no DI framework, proportionate to the app size.
- **Domain layer stays Android-free**; expected open failures use `BookOpenException`.
- **Format routing**: `BookOpener` sniffs the file signature and picks the reader (`MainActivity` for FB2, `PdfReaderActivity` for PDF), so every entry point (library, file picker, last-opened book) handles both.
- **PDF stack**: the reader depends on a small `PdfEngine`/`PdfSession` interface (Pdfium implementation, fakes in tests), `PdfViewport` holds all zoom/pan/scroll math as plain testable Kotlin, and `PdfPageView` renders only the visible region of a page into a viewport-sized bitmap on a background thread.
- **Rendering**: `ReaderView` allocates nothing per frame, resolves theme colors once, and exposes `performClick()` for accessibility services.
- **Modern platform APIs**: `WindowCompat`/`WindowInsetsControllerCompat` for immersive mode, AndroidX KTX helpers, `DateTimeFormatter`, `ActivityResultContracts` (including `CreateDocument`), and no obsolete `SDK_INT` checks or dead `READ_EXTERNAL_STORAGE`/`WAKE_LOCK` permissions.

### Performance
- **Image dithering**: Floyd-Steinberg runs on a flat `IntArray` with batch `getPixels`/`setPixels`, on covers already downsampled to at most ~600x900.
- **RecyclerView**: `ListAdapter` + `DiffUtil`, lifecycle-bound cover loading, per-size `LruCache` entries and job cancellation on recycle.
- **Dynamic anti-aliasing**: enabled on regular screens, disabled when E-Ink optimization is active.

## Known Limitations & Roadmap
- **PDF text**: selection works per word (long-press); selecting a phrase or paragraph is not implemented yet. Scanned documents have no text layer, so selection and search find nothing in them (OCR is out of scope).
- **APK size**: Pdfium adds about 18 MB of native libraries (four ABIs) to the universal APK; restricting release builds to `arm64-v8a` and `armeabi-v7a` would cut most of that for E-Ink devices.
- **Pdfium binding version**: `pdfiumandroid` 2.0.1 is pinned on purpose — 2.0.3 is built with Kotlin 2.4 metadata, which the Kotlin 2.2 compiler bundled with AGP cannot read. Upgrade both together.
- **Target SDK 34**: raising it to the latest level enforces edge-to-edge rendering (the library screen needs window-inset handling) and predictive back; do it together with a visual pass on real devices.
- **All-files access** is still used for storage scanning and custom fonts in `/sdcard/Fonts`; migrating to a Storage Access Framework folder picker would remove that permission.
- Covers created by older versions may still sit in the cache dir; they are not migrated and fall back to a placeholder if the system clears the cache (rescanning or reopening the book restores them).
- Kotlin is provided by AGP's built-in Kotlin support (2.2.x); no separate Kotlin Gradle plugin is applied.

## Technical Architecture

The project follows **Clean Architecture** principles to ensure maintainability and testability:

- **Domain Layer**: Contains business logic, models (`Book`), and repository interfaces.
- **Data Layer**: Implements repositories using Room Database for persistence and a custom FB2 parser.
- **Logic Layer**: Handles complex tasks like asynchronous pagination, storage scanning, and hardware-specific E-Ink management.
- **UI Layer**: MVVM pattern throughout — `LibraryViewModel` and `ReaderViewModel`, both `StateFlow`-driven, with ViewBinding-based Activities.
- **Dependency Injection**: A single lightweight `AppContainer` (held by `VellumApplication`) is the composition root for both Activities — no DI framework, proportionate to the app's size.

## Technical Stack

- **Language**: Kotlin 2.2 (compiled via AGP's built-in Kotlin support — no separate Kotlin Gradle plugin), KSP 2.3
- **Concurrency**: Kotlin Coroutines & Flow
- **Persistence**: Room 2.8 (SQLite) with exported schemas
- **PDF**: Pdfium via `io.legere:pdfiumandroid`
- **UI**: Native Android Canvas + StaticLayout (No WebView), ViewBinding for dialogs/Activities
- **Testing**: JUnit4 + Robolectric + kotlinx-coroutines-test — parser, pagination, library list logic, settings, backup, scanner and both ViewModels (including an end-to-end open → paginate → restyle run)
- **Build System**: Gradle 9.7 + Android Gradle Plugin 9.4 with a Version Catalog for centralized dependency management; dependencies tracked against current stable releases
- **Min SDK**: 30 (Android 11)
- **Target SDK**: 34

## How to Use

1. Launch **Vellum**.
2. The **Library** shows your recently opened books.
3. Tap **Scan Storage** to search targeted folders (e.g. `Books`, `Download`, `Documents`) or the entire device for FB2 books. (Requires granting "All Files Access" permission on Android 11+).
4. Tap **Open New Book** to manually select a specific FB2, ZIP or PDF file using the system file picker.
5. Filter or group your library by clicking the dropdown menus (e.g., group by Author or Series, or filter by Finished books).
6. Open the **⋮** menu in the library and tap **Backup** to save your reading progress and preferences to a file of your choice, or **Restore** to pick a backup JSON file.
7. Tap the center of the reader screen to open the **Menu**.
8. Long-press any book in the **Library** to prompt options to delete only from library or physically delete from device.
8. Use **Volume Buttons** or **Screen Edges** to navigate through pages.

---
*Created with focus on simplicity and reading comfort.*
