# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Logseq plugin that syncs notes from Logseq to Anki flashcards. It supports multiple card types (cloze, multiline, swift arrow, image occlusion), converts Logseq markdown/org mode to HTML for Anki cards, and manages bidirectional sync with advanced features like dependency hashing for change detection.

## Build and Development Commands

**Package Manager:** This project uses `pnpm` (enforced via preinstall script).

**Development:**
- `pnpm dev` - Start development server with hot reload
- `pnpm build` - Build production plugin (sets NODE_OPTIONS for 4GB memory)
- `pnpm test` - Run tests with Vitest

**Testing:**
- Run single test file: `pnpm test <path-to-test-file>`
- Test files are located in `tests/` directory with `.test.ts` extension

## Architecture Overview

### Core Sync Flow

The main sync process is orchestrated by `LogseqToAnkiSync` class in `src/syncLogseqToAnki.ts`:

1. **Scanning Phase** - Scans Logseq graph for different note types (ClozeNote, SwiftArrowNote, ImageOcclusionNote, MultilineCardNote)
2. **Categorization** - Determines which notes need creation, update, or deletion based on Anki IDs and dependency hashes
3. **User Confirmation** - Shows user a dialog with pending operations via `SyncSelectionDialog`
4. **Sync Execution** - Creates/updates/deletes notes in Anki via `LazyAnkiNoteManager`
5. **Asset Management** - Syncs media files (images, PDFs) referenced in cards
6. **Result Display** - Shows sync summary via `SyncResultDialog`

### Note Type System

All note types inherit from the abstract `Note` class (`src/notes/Note.ts`):

- **ClozeNote** (`src/notes/ClozeNote.ts`) - Handles cloze deletion cards with `{{c1::text}}` syntax
- **MultilineCardNote** (`src/notes/MultilineCardNote.ts`) - Front/back cards split by `#card` delimiters
- **SwiftArrowNote** (`src/notes/SwiftArrowNote.ts`) - Bidirectional cards using arrow syntax
- **ImageOcclusionNote** (`src/notes/ImageOcclusionNote.ts`) - Image-based occlusion cards

Each note type implements:
- `getNotesFromLogseqBlocks()` - Static method to extract notes from Logseq
- `getClozedContentHTML()` - Convert note content to HTML for Anki
- `initLogseqOperations()` - Setup Logseq UI elements and event handlers

### Dependency Hashing & Change Detection

The plugin uses sophisticated hashing (via `NoteHashCalculator` in `src/notes/NoteHashCalculator.ts`) to detect when cards need updates. The dependency hash includes:
- Card content HTML
- Referenced assets (images, PDFs)
- Deck assignment
- Breadcrumb/navigation context
- Tags
- Extra fields
- Dependencies on referenced blocks/pages via `getLogseqContentDirectDependencies`

This allows skipping updates when neither the card nor its dependencies have changed.

### Logseq Integration Layer

**LogseqProxy** (`src/logseq/LogseqProxy.ts`) - Wrapper around Logseq API with caching and error handling
- Caches block/page queries to minimize API calls
- Provides silent page creation (`createPageSilentlyIfNotExists`)
- Handles DB vs file-based graphs (DB graphs not yet supported)

**LogseqToHtmlConverter** (`src/logseq/LogseqToHtmlConverter.ts`) - Converts Logseq markdown/org mode to HTML
- Handles block references, page references, macros
- Processes embedded content (images, PDFs, videos)
- Supports syntax highlighting via highlight.js

**BlockContentParser** (`src/logseq/BlockContentParser.ts`) - Parses special syntax in blocks
- Extracts card delimiters (`#card`, `#card-reverse`)
- Handles cloze syntax
- Processes hints and extra content

### Anki Communication

**AnkiConnect** (`src/anki-connect/AnkiConnect.ts`) - Low-level AnkiConnect API wrapper
- Communicates with Anki via HTTP on port 8765
- Handles permission requests
- Provides CRUD operations for notes

**LazyAnkiNoteManager** (`src/anki-connect/LazyAnkiNoteManager.ts`) - Batches operations for performance
- Queues note operations (add/update/delete)
- Batches asset storage
- Executes operations in bulk to minimize round trips

### UI Components

React-based UI components in `src/ui/`:
- **ProgressNotification** - Shows sync progress
- **SyncSelectionDialog** - User selects which operations to perform
- **SyncResultDialog** - Displays sync results
- **OcclusionEditor** - Image occlusion editor using Fabric.js
- **Modal system** - Reusable modal/dialog components

### Addon System

Addons (`src/addons/`) extend plugin functionality:
- **PreviewInAnki** - Context menu to preview cards in Anki
- **HideOcclusionData** - Hides occlusion metadata from Logseq UI
- **LogseqAnkiFeatureExplorer** - Interactive feature discovery UI

Addons register via `AddonRegistry` and implement the `Addon` interface with `init()` method.

### Settings System

Settings (`src/settings.ts`) are managed via Logseq's settings API and include:
- Default deck configuration
- Breadcrumb display options
- Parent content inclusion
- Namespace-as-deck behavior
- Debug flags

## Important Implementation Details

**Graph-Specific Models:** Each Logseq graph gets its own Anki model (note type) named `{GraphName}Model`. This isolates cards from different graphs.

**UUID Persistence:** Plugin adds `id:: {uuid}` property to blocks to ensure UUIDs persist across re-indexing.

**DB Graph Detection:** Current version checks for DB graphs via `LogseqProxy.App.checkCurrentIsDbGraph()` and shows error (DB support is in development on `db` branch).

**Template System:** Card templates are in `src/templates/` with separate front/back/shared JavaScript files that get injected into Anki cards.

**Vite Plugins:** Custom Vite plugins handle:
- `staticFileSyncTransformPlugin` - Inlines `fs.readFileSync` calls at build time
- `bundleJSStringPlugin` - Bundles `.js?string` imports for template injection

## Common Patterns

**Block/Page Property Access:** Use `getLogseqBlockPropSafe()` utility instead of direct lodash `_.get()` to safely access nested properties with proper type handling.

**Tag Matching:** Use `NoteUtils.matchTagNamesWithTagIds()` to resolve tag IDs to tag names when checking for specific tags.

**Namespace Handling:** Use `splitNamespace()` utility to convert Logseq's `/` namespace separators to Anki's `::` deck separators.

**Parent Block Traversal:** When traversing up block/page hierarchy, always check for null and use `_.get(block, 'parent.id', null)` pattern to safely access parent references.

## Testing

Tests are configured via `vitest.config.ts`:
- Include pattern: `**/*.test.ts`
- Excludes: `logseq-dev-plugin` and `node_modules`
- Aliases `@logseq/libs` to `sass` for testing without Logseq runtime

Current test coverage includes:
- `tests/compareAnswer/` - Answer comparison logic
- `tests/converter/` - HTML conversion tests
