# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Unciv is an open-source Android and Desktop remake of Civilization V, built with LibGDX and Kotlin. It's a moddability-focused 4X strategy game with support for multiple platforms (Android, Desktop, Server).

## Build Commands

### Desktop Development
- **Run the game**: `./gradlew desktop:run`
- **Build JAR**: `./gradlew desktop:dist` (output: `desktop/build/libs/Unciv.jar`)
- **Debug mode**: `./gradlew desktop:debug`

### Server
- **Run server**: `./gradlew server:run`
- **Build server JAR**: `./gradlew server:dist` (output: `server/build/libs/UncivServer.jar`)

### Testing
- **Run all tests**: `./gradlew --no-build-cache cleanTest test tests:test`
- **Run specific test**: `./gradlew tests:test --tests "com.unciv.logic.*"`
- **Run tests for a specific class**: `./gradlew tests:test --tests "ClassName"`

### Code Quality
- **Compile and build classes**: `./gradlew classes`
- **Run code checks**: `./gradlew check`
- **Detekt (linting)**:
  - Warnings: `PATH/TO/DETEKT/detekt-cli --parallel --report html:detekt/reports.html --config .github/workflows/detekt_config/detekt-warnings.yml`
  - Errors: `PATH/TO/DETEKT/detekt-cli --parallel --report html:detekt/reports.html --config .github/workflows/detekt_config/detekt-errors.yml`

### Android (requires ANDROID_HOME or sdk.dir in local.properties)
- Android build only available if Android SDK is configured
- The android module is conditionally included based on SDK availability

## Rapid Development & Hot Reload

### Continuous Build Mode
Gradle supports automatic recompilation when source files change:
```bash
./gradlew desktop:run --continuous
```
**Note**: This recompiles automatically but does NOT restart the application - you still need to manually restart to see changes.

### FasterUIDevelopment (Recommended for UI Work)
For rapid UI component iteration, use the `FasterUIDevelopment` tool:

**Location**: `tests/src/com/unciv/dev/FasterUIDevelopment.kt`

**How to use**:
1. Edit the `DevElement.createDevElement()` method to return your UI component:
   ```kotlin
   fun createDevElement() {
       actor = YourCustomUIComponent()  // Replace the default label
   }
   ```

2. Run the main method:
   - **Option A**: Click the green arrow next to `main()` in Android Studio
   - **Option B**: Create a Run Configuration:
     - Main class: `com.unciv.dev.FasterUIDevelopment`
     - Module classpath: `Unciv.tests.test`
     - Working directory: `android/assets`

**Benefits**:
- Starts in ~2-3 seconds (vs 10+ for full game)
- Shows only your component with an orange border
- Toggle Scene2D debug mode with middle mouse button
- Window size persisted separately from main game
- No need to navigate through game menus to see your UI

**Workflow**: Edit code → Rerun FasterUIDevelopment → See changes instantly

### Development Workflow Recommendations
- **UI component work**: Use `FasterUIDevelopment` for fastest iteration
- **Game logic changes**: Use `./gradlew desktop:run --continuous` + manual restart
- **Full integration testing**: Regular `./gradlew desktop:run`

## Project Structure

### Module Organization
- **core/**: 99% of the codebase - all platform-independent game logic and UI
- **desktop/**: Desktop launcher and platform-specific code
- **android/**: Android launcher, game images, and assets (used by all platforms)
- **server/**: UncivServer multiplayer host
- **tests/**: Unit tests (run with `./gradlew tests:test`)

### Core Architecture (core/src/com/unciv/)

#### Game State Hierarchy
The game state is serializable and follows this structure:
- **GameInfo** (root) - The entire game state
  - **CivilizationInfo** - A player/civ in the game
    - **CityInfo** - A specific city
  - **TileMap** - The game map
    - **TileInfo** - Individual map tiles
      - **MapUnit** - Units on the map
  - **RuleSet** - Game rules (NOT serialized, loaded from JSON)

Key directories:
- `logic/` - Game state and business logic
  - `logic/civilization/` - Civilization management, diplomacy, policies, techs
  - `logic/city/` - City management, construction, population
  - `logic/map/` - Map generation, tile management, units
  - `logic/automation/` - AI and automation systems
  - `logic/battle/` - Combat calculations
- `ui/` - All UI screens and components
  - `ui/screens/` - Main game screens (WorldScreen, CityScreen, etc.)
  - `ui/components/` - Reusable UI components
- `models/` - Data models and ruleset objects
- `utils/` - Utility functions and helpers
- `json/` - JSON serialization

#### Key Classes
- **UncivGame** - Main UI entry point
- **GameInfo** - Root of game state, contains civilizations, map, and ruleset
- **CivilizationInfo** - Represents a player with cities, policies, techs, diplomacy
- **CityInfo** - Individual city with population, constructions, stats
- **TileMap/TileInfo** - Map structure and individual tiles
- **MapUnit** - Specific unit instances (vs BaseUnit which is the unit type)
- **RuleSet** - Contains all game objects (loaded from JSON, not serialized)

### Ruleset System
Game content (units, buildings, nations, etc.) is defined in JSON files at `android/assets/jsons/`:
- `Civ V - Vanilla/` - Base game ruleset
- `Civ V - Gods & Kings/` - G&K expansion content
- `Tutorials.json` - Tutorial content
- `TileSets/` - Tileset definitions
- `translations/` - Translation files

Mods can extend or replace the ruleset. Base ruleset mods set `"isBaseRuleset": true` in ModOptions.json.

## Important Technical Details

### Purity Plugin
The project uses a custom "purity plugin" (`io.github.yairm210.purity-plugin`) to enforce functional purity in code. Well-known pure/readonly functions are configured in the root `build.gradle.kts`.

### Dependencies
- **Kotlin**: 2.1.21
- **LibGDX**: 1.14.0 (game framework)
- **Ktor**: 3.2.3 (networking for multiplayer)
- **Coroutines**: 1.8.1
- **Java**: Target JDK 21 (source compatibility), compile to Java 8 bytecode

### NextTurn Process
- GameInfo is cloned for each turn for thread safety and multiplayer reproducibility
- NextTurn runs in a separate thread to keep UI responsive
- Prevents conflicts during rendering and serialization
- Essential for multiplayer where entire game state is transmitted

### Working Directory
When running desktop or server, the working directory must be set to `android/assets/` (this is configured in build files).

### Translation System
All user-facing strings must be translatable. New strings may need to be added to `android/assets/jsons/translations/template.properties`. See the "Translation generation - for developers" section in docs for details.

## Development Workflow

### Testing Strategy
- Tests mirror the core package structure in `tests/src/com/unciv/`
- Working directory for tests is set to `../android/assets`
- Use headless backend for LibGDX in tests

### Assets and Resources
- Assets are in `android/assets/` and are bundled into the desktop JAR
- SaveFiles and mods directories should be excluded from version control
- When testing, mark `android/assets/SaveFiles` and `android/assets/mods` as excluded in IDE

### Common Pitfalls
- Forgetting to set working directory to `android/assets/` causes file-not-found errors
- Unit vs MapUnit confusion: MapUnit is an instance, BaseUnit is the type
- The RuleSet is NOT part of game state serialization - it's loaded from JSON
- Transient parent references in state tree allow bidirectional traversal

## Version Information
- Current version defined in `buildSrc/src/main/kotlin/BuildConfig.kt`
- Version format: `appVersion`, `appCodeNumber`, and `identifier`
