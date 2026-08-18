# Copilot Instructions for Lilac Anti-Cheat

## Project Overview
This repository contains **Lilac (Little Anti-Cheat)**, a free and open-source anti-cheat system for Source engine games, built as a SourcePawn plugin for SourceMod. Lilac provides real-time detection of various cheats including aimbots, angle exploits, bunnyhopping, chat clearing, and more across multiple Source engine games.

**Primary Purpose**: Real-time cheat detection and automated banning for Source game servers
**Target Games**: Team Fortress 2, Counter-Strike: Source, Left 4 Dead 2, Left 4 Dead, Day of Defeat: Source

## Technical Environment

### Core Technologies
- **Language**: SourcePawn (Source engine scripting language)
- **Platform**: SourceMod 1.11+ (minimum 1.12+ recommended)
- **Compiler**: SourcePawn compiler (spcomp), installed via `rumblefrog/setup-sp`
- **Build System**: Native GitHub Actions workflow (`.github/workflows/ci.yml`), no external build tool
- **CI/CD**: GitHub Actions with automated building and releases

### Development Dependencies
- SourceMod 1.12 development headers (installed by the CI workflow via `rumblefrog/setup-sp`)
- Game-specific includes (TF2, CS:S, L4D2, etc.) that ship with SourceMod

## Architecture & Structure

### Modular Design
The plugin follows a modular architecture with the main file including specialized detection modules:

```
addons/sourcemod/scripting/
├── lilac.sp                    # Main plugin file and orchestrator
├── include/
│   ├── lilac.inc              # Public API and forwards for other plugins
│   └── convar_class.inc       # ConVar wrapper with methodmap
└── lilac/                     # Detection modules
    ├── lilac_globals.sp       # Global definitions and constants
    ├── lilac_config.sp        # Configuration management  
    ├── lilac_database.sp      # Database logging (async SQL)
    ├── lilac_aimbot.sp        # Aimbot detection algorithms
    ├── lilac_aimlock.sp       # Aimlock detection
    ├── lilac_angles.sp        # Invalid angle detection
    ├── lilac_bhop.sp          # Bunnyhopping detection
    ├── lilac_backtrack.sp     # Backtrack exploit detection
    ├── lilac_convar.sp        # ConVar validation
    ├── lilac_lerp.sp          # Interpolation exploit detection
    ├── lilac_macro.sp         # Macro/automation detection
    ├── lilac_noisemaker.sp    # Noisemaker spam detection (TF2)
    ├── lilac_ping.sp          # High ping kicking
    ├── lilac_string.sp        # Chat/name validation
    └── lilac_stock.sp         # Utility functions
```

### Key Architecture Principles
1. **Event-driven**: Uses SourceMod's event system for game state monitoring
2. **Asynchronous**: All database operations use async SQL with methodmaps
3. **Game-agnostic**: Detection algorithms adapt to different Source games
4. **Configurable**: Extensive ConVar system for server customization
5. **Extensible**: Public API via forwards for integration with other plugins

## Code Style & Standards

### SourcePawn Conventions (STRICTLY ENFORCED)
```sourcepawn
#pragma semicolon 1           // REQUIRED - All statements must end with semicolon
#pragma newdecls required     // REQUIRED - Use new syntax declarations

// Naming conventions:
int g_iGlobalVariable;        // Global variables: g_ prefix + Hungarian notation
bool clientData[MAXPLAYERS];  // Arrays: descriptive camelCase
void FunctionName()           // Functions: PascalCase
int localVar;                 // Local variables: camelCase
```

### Indentation & Formatting
- **Indentation**: Tabs only (equivalent to 4 spaces)
- **Braces**: K&R style (opening brace on same line)
- **Trailing spaces**: Must be removed
- **Line endings**: Unix LF

### Variable Naming Patterns
```sourcepawn
// Global variables
ConVar g_hCvarEnable;         // ConVar handles: g_h prefix
Database g_dbHandle;          // Database handles: g_db prefix
bool g_bPlayerStatus[MAXPLAYERS]; // Player arrays: g_b/g_i/g_f prefix

// Constants
#define CHEAT_AIMBOT 5        // Constants: ALL_CAPS with underscores
#define MAX_DETECTIONS 10
```

## Build System (Native GitHub Actions)

### Configuration
The project builds directly with `spcomp` in `.github/workflows/ci.yml` — there is no separate build-tool config file:

```yaml
# .github/workflows/ci.yml (build job, simplified)
- uses: rumblefrog/setup-sp@v1.3.1
  with:
    version: "1.12.x"
- working-directory: addons/sourcemod/scripting
  run: spcomp -i include -o ../plugins/lilac.smx lilac.sp
```

### Build Commands
```bash
# Local development build (requires spcomp on PATH)
cd addons/sourcemod/scripting
spcomp -i include -o ../plugins/lilac.smx lilac.sp
```

### CI/CD Pipeline
- **Triggers**: Push, PR, manual dispatch
- **Process**: Build → Package → Release (on tags/master)
- **Artifacts**: Compiled .smx files + translations packaged

## Database & Performance

### Database Requirements
- **ALL SQL queries MUST be asynchronous** - No blocking database calls
- **Use methodmaps** for database operations
- **Escape all user input** to prevent SQL injection
- **Use transactions** for multi-query operations
- **Connection handling**: Auto-reconnect on failure

```sourcepawn
// CORRECT: Async database usage
Database.Connect(OnDatabaseConnected, "lilac");
db.Query(OnQueryComplete, "SELECT * FROM bans WHERE steamid = '%s'", steamid);

// WRONG: Never use synchronous queries in production
// SQL_FastQuery(db, query); // This will block the game server!
```

### Performance Considerations
- **Minimize timer usage** - Prefer event-driven programming
- **Cache expensive calculations** - Store results, don't recalculate
- **Optimize hot paths** - Functions called every tick/frame must be efficient
- **Memory management**: Use `delete` for cleanup, avoid `.Clear()` on StringMaps/ArrayLists

```sourcepawn
// CORRECT: Efficient memory management
delete playerMap;
playerMap = new StringMap();

// WRONG: Creates memory leaks
playerMap.Clear(); // Don't use this!
```

## Anti-Cheat Specific Guidelines

### Detection Module Development
1. **Statistical validation**: Require multiple positive detections before action
2. **False positive mitigation**: Include edge case handling and thresholds
3. **Game compatibility**: Test across all supported Source games
4. **Performance impact**: Detection algorithms must not affect server performance

### Security Considerations
- **Obfuscation resistance**: Assume cheaters can read the source code
- **Detection signatures**: Avoid hardcoded patterns that can be easily bypassed
- **Timing attacks**: Use randomized delays and detection windows
- **Log sanitization**: Never log sensitive player data unnecessarily

### Testing Requirements
- **False positive testing**: Use legitimate gameplay scenarios
- **Cross-game compatibility**: Test on TF2, CS:S, L4D2, etc.
- **Performance profiling**: Monitor tick rate impact during heavy gameplay
- **Database stress testing**: Verify async operations under load

## Configuration Management

### ConVar System
```sourcepawn
// ConVar creation pattern
hcvar[CVAR_ENABLE] = new Convar("lilac_enable", "1",
    "Enable Little Anti-Cheat.",
    FCVAR_PROTECTED, true, 0.0, true, 1.0);
```

### Configuration Files
- **Server operators**: Provide clear documentation for all ConVars
- **Default values**: Choose safe defaults that work for most servers
- **Migration**: Handle config updates gracefully

## Translation & Localization

### Translation Requirements
- **All user-facing messages** must use translation phrases
- **File location**: `addons/sourcemod/translations/lilac.phrases.txt`
- **Multiple languages**: Currently supports 15+ languages
- **Fallback**: Always provide English fallback

```sourcepawn
// CORRECT: Using translations
PrintToChat(client, "%t", "LILAC_Detection_Message", cheatName);

// WRONG: Hardcoded English text
PrintToChat(client, "Cheat detected!");
```

## Integration & API

### Public API (lilac.inc)
```sourcepawn
// Forward for other plugins
forward void lilac_cheater_detected(int client, int cheat);

// Cheat type constants (DO NOT MODIFY - used by other plugins)
#define CHEAT_AIMBOT     5
#define CHEAT_ANGLES     0
// ... other constants
```

### External Plugin Support
- **SourceBans++**: Automatic ban integration
- **Material-Admin**: Alternative ban system
- **SourceIRC**: IRC notification support
- **Custom integrations**: Via public forwards

## Testing & Validation

### Required Testing
1. **Compilation test**: Must compile without warnings
2. **Game compatibility**: Test on primary supported games
3. **Database connectivity**: Verify async SQL operations
4. **Memory leak testing**: Use SourceMod profiler
5. **Performance impact**: Monitor server tick rate

### Development Workflow
1. **Local testing**: Use development server with debug logging
2. **Code review**: Focus on performance and security implications  
3. **Staged deployment**: Test on low-population servers first
4. **Monitoring**: Watch for false positives and performance issues

## Common Patterns & Anti-Patterns

### Recommended Patterns
```sourcepawn
// Efficient player data access
bool IsPlayerValid(int client) {
    return (client > 0 && client <= MaxClients && IsClientInGame(client));
}

// Proper timer cleanup
if (timer_handle != null) {
    delete timer_handle;
    timer_handle = null;
}
```

### Anti-Patterns to Avoid
```sourcepawn
// WRONG: Inefficient loops in tick-based functions
for (int i = 1; i <= MaxClients; i++) {
    // Heavy processing every tick - BAD!
}

// WRONG: Blocking database calls
SQL_FastQuery(db, query); // Never do this!

// WRONG: Memory leaks
stringMap.Clear(); // Use delete instead!
```

## Documentation Requirements

### Code Documentation
- **Complex algorithms**: Document detection logic and thresholds
- **Game-specific code**: Explain why different games need different handling
- **Performance considerations**: Note any optimization decisions
- **Security implications**: Document anti-bypass measures

### No Unnecessary Comments
- **Don't document obvious code** - The codebase avoids header comments
- **Focus on WHY, not WHAT** - Explain reasoning, not syntax
- **Update with changes** - Keep documentation current

## Emergency Procedures

### Critical Issues
1. **False positive epidemic**: Disable detection via ConVar immediately
2. **Performance degradation**: Check recent changes to hot paths
3. **Database connectivity**: Graceful degradation without logging
4. **Compatibility breaks**: Game updates may require detection adjustments

### Rollback Strategy
- **ConVar-based toggles**: All detections can be disabled without restart
- **Modular design**: Individual detection modules can be isolated
- **Configuration backup**: Maintain known-good configurations

---

## Quick Reference

### Essential Commands
```bash
# Build plugin (from addons/sourcemod/scripting)
spcomp -i include -o ../plugins/lilac.smx lilac.sp

# Test translations
sm plugins load lilac
```

### Key Files for Common Tasks
- **Adding new detection**: Create module in `lilac/` directory
- **Modifying API**: Update `include/lilac.inc`
- **Database changes**: Modify `lilac/lilac_database.sp`
- **Configuration**: Update `lilac/lilac_config.sp`

This project requires careful attention to performance, security, and accuracy - the real-time nature of anti-cheat detection means code quality directly impacts gameplay experience.