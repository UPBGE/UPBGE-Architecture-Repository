# KX_SaveSystem Architecture Documentation

## Overview

The KX_SaveSystem is a pure C++ save/load system for UPBGE designed to work with or without Python. It provides robust, efficient, and portable game state persistence using industry-standard serialization formats.

## Design Goals

1. **Python-Independent Core**: All core functionality implemented in pure C++ to support Python-free builds (Android)
2. **Performance First**: Fast serialization with MessagePack, efficient compression with zlib
3. **Minimal Blender Impact**: Isolated in gameengine folder, no modifications to upstream Blender code
4. **Optional Python Bindings**: Full Python API available when compiled with `WITH_PYTHON`
5. **Production Ready**: Atomic file operations, checksums, encryption, version migration support

## Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│         Python Bindings (Optional - WITH_PYTHON)        │
│  PyObject* ←→ SaveData conversion, Python API methods   │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│              C++ Core (Always Available)                │
│  SaveData container, KX_SaveSystem singleton            │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                  Serialization Layer                     │
│  MessagePack (binary) ←→ JSON (metadata/debug)          │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│              Compression & Encryption                    │
│  zlib (compression) + ChaCha8 (encryption)              │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   File I/O Layer                        │
│  BLI utilities (portable file operations)               │
└─────────────────────────────────────────────────────────┘
```

## Core Components

### 1. SaveData (C++ Container)

**Purpose**: Type-safe wrapper around `nlohmann::json` for game state storage.

**Key Features**:
- Type-safe setters/getters for common types (bool, int64, float, string, vector)
- Nested data support via `SetData()`/`GetData()`
- Default value support on getters
- Key enumeration and existence checking

**Usage Example**:
```cpp
SaveData player;
player.SetString("name", "Hero");
player.SetInt("level", 5);
player.SetFloat("health", 100.0);
player.SetVector("position", {10.5, 0.0, 5.2});

SaveData inventory;
inventory.SetInt("gold", 500);
player.SetData("inventory", inventory);

// Retrieval with defaults
int level = player.GetInt("level", 1);
std::string name = player.GetString("name", "Unknown");
```

### 2. KX_SaveSystem (Singleton)

**Purpose**: Main interface for all save/load operations.

**Responsibilities**:
- Slot management (create, delete, list, validate)
- Serialization orchestration (MessagePack, JSON)
- Compression/decompression with zlib
- Optional encryption with ChaCha8
- Metadata handling
- Version migration support
- Error reporting

**Initialization**:
```cpp
KX_SaveSystem *sys = KX_SaveSystem::GetInstance();
sys->SetSavePath("//saves/");
sys->SetGameVersion("1.0.0");
```

### 3. SaveMetadata

**Purpose**: Store metadata about save files separately from game state.

**Contents**:
- Save system version
- Game version (for migration)
- Timestamp (Unix time)
- Playtime counter
- Optional screenshot (base64)
- Custom metadata (key-value map)

**Serialization**: JSON format for human readability

### 4. File Format

#### Binary Save Format (MessagePack)

```
┌──────────────────────────────────────────┐
│  SaveFileHeader (32 bytes)               │
│  ┌────────────────────────────────────┐  │
│  │ magic:          0x55504247 (UPBG) │  │
│  │ version:        1                  │  │
│  │ original_size:  uncompressed size  │  │
│  │ flags:          COMPRESSED|ENCRYPT │  │
│  │ checksum:       Adler32            │  │
│  │ reserved:       0                  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│  Compressed/Encrypted Data               │
│  (MessagePack serialized SaveData)       │
└──────────────────────────────────────────┘
```

**Endianness**: Little-endian (converted on big-endian platforms)

**Checksum**: Adler32 on uncompressed data (fast, good error detection)

**Flags**:
- `SAVE_FLAG_COMPRESSED (1 << 0)`: Data is zlib compressed
- `SAVE_FLAG_ENCRYPTED (1 << 1)`: Data is ChaCha8 encrypted

#### Directory Structure

```
//saves/
  ├── slot1/
  │   ├── save.msgpack      # Binary save (primary)
  │   ├── save.json         # JSON save (optional, debug)
  │   ├── metadata.json     # Metadata (always present)
  │   └── thumbnail.png     # Screenshot (optional)
  ├── slot2/
  │   ├── save.msgpack
  │   ├── metadata.json
  │   └── thumbnail.png
  └── autosave/
      ├── save.msgpack
      └── metadata.json
```

## Serialization Flow

### Save Operation

```
SaveData (C++) 
  ↓ SerializeToMsgPack()
MessagePack bytes
  ↓ EncryptDecrypt() [if enabled]
Encrypted bytes
  ↓ CompressData()
Header + Compressed bytes
  ↓ Atomic write (temp → rename)
save.msgpack file
```

### Load Operation

```
save.msgpack file
  ↓ Read entire file
Header + Compressed bytes
  ↓ DecompressData()
Encrypted bytes (or plain)
  ↓ EncryptDecrypt() [if needed]
MessagePack bytes
  ↓ DeserializeFromMsgPack()
SaveData (C++)
  ↓ [Optional] ApplyMigrations()
Final SaveData
```

## API Reference

### C++ API

#### Save Operations

```cpp
// Basic save
SaveData data;
data.SetInt("score", 1000);
sys->SaveGame("slot1", data);

// Save with options
SaveOptions opts;
opts.compress = true;
opts.encrypt = true;
opts.encryption_key = "my_secret_key";
opts.custom_metadata["chapter"] = "prologue";
sys->SaveGame("slot1", data, opts);

// Partial save (only specified keys)
opts.partial_keys = {"score", "level"};
sys->SaveGame("slot1", data, opts);
```

#### Load Operations

```cpp
// Basic load
SaveData loaded = sys->LoadGame("slot1");
if (sys->GetLastErrorType() != SAVE_ERROR_NONE) {
    std::cout << "Error: " << sys->GetLastErrorMessage() << std::endl;
}

// Load with options
LoadOptions opts;
opts.validate = true;
opts.auto_migrate = true;
opts.decrypt = true;
opts.encryption_key = "my_secret_key";
SaveData loaded = sys->LoadGame("slot1", opts);

// Partial load
opts.partial_keys = {"score", "level"};
SaveData partial = sys->LoadGame("slot1", opts);
```

#### Slot Management

```cpp
// List all saves
std::vector<SaveSlotInfo> saves = sys->ListSaves();
for (const auto &info : saves) {
    std::cout << "Slot: " << info.slot_name << std::endl;
    std::cout << "Valid: " << info.is_valid << std::endl;
    std::cout << "Game Version: " << info.metadata.game_version << std::endl;
}

// Check existence
if (sys->SaveExists("slot1")) {
    // Load or overwrite
}

// Validate save file
if (sys->ValidateSave("slot1")) {
    // File is readable
}

// Delete save
sys->DeleteSave("slot1");
```

#### Version Migration

```cpp
// Register migration callback
sys->RegisterMigration("1.0.0", "2.0.0", 
    [](const SaveData &old_data, const std::string &from, const std::string &to) {
        SaveData new_data = old_data;
        
        // Migrate data structure
        if (old_data.HasKey("old_score")) {
            int old_score = old_data.GetInt("old_score");
            new_data.SetInt("new_score", old_score * 2);
            new_data.Remove("old_score");
        }
        
        return new_data;
    }
);
```

### Python API (Optional - WITH_PYTHON)

```python
import SaveSystem

# Configuration
SaveSystem.setSavePath("//saves/")
SaveSystem.setGameVersion("1.0.0")

# Save
data = {
    "player_name": "Hero",
    "level": 5,
    "score": 1000,
    "inventory": {
        "gold": 500,
        "items": ["sword", "shield"]
    }
}

options = {
    "compress": True,
    "encrypt": True,
    "encryption_key": "my_secret",
    "format": 0  # SAVE_FORMAT_BINARY
}

SaveSystem.saveGame("slot1", data, options)

# Load
loaded = SaveSystem.loadGame("slot1", {"decrypt": True, "encryption_key": "my_secret"})
print(loaded["player_name"])  # "Hero"

# List saves
saves = SaveSystem.listSaves()
for save_info in saves:
    print(f"Slot: {save_info['slot']}")
    print(f"Valid: {save_info['is_valid']}")
    print(f"Timestamp: {save_info['metadata']['timestamp']}")

# Delete
SaveSystem.deleteSave("slot1")

# Check existence
if SaveSystem.saveExists("autosave"):
    data = SaveSystem.loadGame("autosave")
```

## Dependencies

### Required (C++ Core)

1. **msgpack-c** (MIT License)
   - C++ MessagePack serialization
   - Efficient binary format
   - Header-only or compiled library

2. **nlohmann/json** (MIT License)
   - JSON for C++
   - Header-only library
   - Used for metadata and JSON saves

3. **zlib** (zlib License)
   - Already included in Blender
   - Compression/decompression
   - Fast and well-tested

4. **BLI (Blender Libraries)**
   - Portable file operations
   - Path manipulation
   - Already available in UPBGE

### Optional (Python Bindings)

5. **Python.h**
   - Only when compiled with `WITH_PYTHON`
   - Provides Python API

## Implementation Details

### Thread Safety

**Current Status**: Not thread-safe (singleton without mutex)

**Recommendations**:
- Use only from main game thread
- For multi-threaded access, add `std::mutex` to singleton
- Atomic file operations prevent corruption from external interference

### Error Handling

**Strategy**: Error codes + message strings

```cpp
SaveData data = sys->LoadGame("slot1");
if (sys->GetLastErrorType() != SAVE_ERROR_NONE) {
    SaveErrorType error = sys->GetLastErrorType();
    std::string msg = sys->GetLastErrorMessage();
    
    switch (error) {
        case SAVE_ERROR_NOT_FOUND:
            // Handle missing file
            break;
        case SAVE_ERROR_CORRUPTED:
            // Handle corrupted save
            break;
        case SAVE_ERROR_VERSION_MISMATCH:
            // Handle version incompatibility
            break;
        // ... other cases
    }
}
```

**Error Types**:
- `SAVE_ERROR_NONE`: No error
- `SAVE_ERROR_IO`: File I/O error
- `SAVE_ERROR_CORRUPTED`: Data corruption detected
- `SAVE_ERROR_VERSION_MISMATCH`: Version incompatibility
- `SAVE_ERROR_NOT_FOUND`: Save file not found
- `SAVE_ERROR_ENCRYPTION`: Encryption/decryption error

### Atomic File Operations

**Pattern**: Write to temporary file → rename to final

```cpp
std::string temp_path = path + ".tmp";

// Write to temp file
std::ofstream file(temp_path, std::ios::binary);
file.write(data, size);
file.flush();
file.close();

// Atomic rename (OS-level operation)
BLI_rename(temp_path.c_str(), path.c_str());
```

**Benefits**:
- Prevents corruption if write interrupted
- Never leaves partially-written save files
- Previous save remains valid until new save completes

### Encryption: ChaCha8

**Algorithm**: ChaCha8 (8 rounds)
- Faster than ChaCha20
- Still cryptographically secure
- Stream cipher (XOR-based)

**Key Derivation**: Simple padding/truncation (32 bytes)
- For production: Use PBKDF2 or similar
- Current implementation for basic obfuscation

**Nonce**: Fixed zero array
- Consider using per-slot unique nonce (hash of slot name)

### Compression: zlib

**Level**: 6 (default balance)
- Configurable via `SaveOptions`
- Level 1 = fastest, Level 9 = best compression

**Bound Calculation**: `compressBound()` for buffer sizing

**Checksum**: Adler32 (faster than CRC32, good enough)

## Performance Characteristics

### Serialization Speed

| Format      | Serialization | Deserialization | Size  |
|-------------|---------------|-----------------|-------|
| MessagePack | ~2x faster    | ~2x faster      | ~30% smaller |
| JSON        | Baseline      | Baseline        | Baseline     |

### Compression Impact

| Level | Speed    | Ratio | Use Case               |
|-------|----------|-------|------------------------|
| 1     | Fastest  | ~2x   | Autosave, frequent     |
| 6     | Balanced | ~4x   | Default, recommended   |
| 9     | Slowest  | ~5x   | Cloud saves, archival  |

### File Size Example

For a typical game state (10,000 objects):

- Raw MessagePack: ~500 KB
- Compressed (level 6): ~125 KB
- With encryption: ~125 KB (no size change)

## Integration with UPBGE

### Logic Bricks Integration

**Action Actuator** could expose:
- Save Current State
- Load Saved State
- Delete Save Slot

**Property Sensor** could check:
- Save Exists
- Save Valid

**Example (C++ in actuator)**:
```cpp
void SaveActuator::Execute() {
    KX_SaveSystem *sys = KX_SaveSystem::GetInstance();
    
    SaveData data;
    // Populate from scene properties
    data.SetInt("level", scene->GetProperty("level"));
    data.SetString("player", scene->GetProperty("player_name"));
    
    sys->SaveGame(m_slot_name, data);
}
```

### Python Integration

**bge.logic module extension**:
```python
import bge

# Access via bge.logic
bge.logic.saveGame("slot1", bge.logic.globalDict)
data = bge.logic.loadGame("slot1")
bge.logic.globalDict.update(data)
```

## Best Practices

### 1. Slot Naming Convention

```
"slot1", "slot2", "slot3"       # Manual saves
"autosave"                       # Auto-save slot
"quicksave"                      # Quick-save slot
"checkpoint_level5"              # Checkpoint saves
```

### 2. Metadata Usage

```cpp
SaveOptions opts;
opts.custom_metadata["chapter"] = "Chapter 3";
opts.custom_metadata["location"] = "Castle";
opts.custom_metadata["difficulty"] = "Hard";
```

### 3. Partial Saves (Optimization)

Only save changed data:
```cpp
SaveOptions opts;
opts.partial_keys = {"player_stats", "inventory"};  // Don't save world state
sys->SaveGame("quicksave", data, opts);
```

### 4. Version Migration Strategy

- Use semantic versioning
- Register migrations for each version jump
- Test migrations with old save files
- Provide fallback for failed migrations

### 5. Error Recovery

```cpp
SaveData data = sys->LoadGame("slot1");
if (sys->GetLastErrorType() == SAVE_ERROR_CORRUPTED) {
    // Try backup
    data = sys->LoadGame("slot1_backup");
}
if (sys->GetLastErrorType() != SAVE_ERROR_NONE) {
    // Start new game
    data = CreateDefaultSaveData();
}
```

## Future Enhancements

### Planned Features

1. **Cloud Save Integration**
   - Abstract file I/O layer
   - Support for HTTP upload/download
   - Sync conflict resolution

2. **Screenshot Capture**
   - Integrate with GPU/render system
   - Automatic thumbnail generation
   - Base64 encoding for metadata

3. **Save File Versioning**
   - Keep multiple versions per slot
   - Rollback capability
   - Automatic backup on overwrite

4. **Streaming for Large Saves**
   - Chunk-based serialization
   - Progress reporting
   - Memory-efficient for huge game states

5. **Compression Profiles**
   - Preset configurations (fast/balanced/small)
   - Per-data-type compression strategies

### Optimization Opportunities

1. **Memory Pooling**
   - Reuse buffers for serialization
   - Reduce allocations in hot paths

2. **Parallel Compression**
   - Multi-threaded zlib compression
   - Chunk-based parallel processing

3. **Delta Encoding**
   - Save only differences from previous save
   - Massive size reduction for sequential saves

## Troubleshooting

### Common Issues

**Issue**: "Failed to create directory"
- **Cause**: Invalid path or permissions
- **Solution**: Check `SetSavePath()` is valid, ensure write permissions

**Issue**: "Checksum verification failed"
- **Cause**: Data corruption or wrong encryption key
- **Solution**: File is corrupted, restore from backup

**Issue**: "MessagePack deserialization failed"
- **Cause**: Invalid data format
- **Solution**: File not a valid save, may be manually edited

**Issue**: "Save version mismatch"
- **Cause**: Loading save from different game version
- **Solution**: Enable `auto_migrate` or register migration callback

### Debug Mode

Enable verbose logging (add to implementation):
```cpp
#define SAVE_SYSTEM_DEBUG 1

#if SAVE_SYSTEM_DEBUG
  #define SAVE_LOG(msg) std::cout << "[SaveSystem] " << msg << std::endl
#else
  #define SAVE_LOG(msg)
#endif
```

## License Considerations

All dependencies use permissive licenses compatible with UPBGE:

- **msgpack-c**: Boost Software License / MIT
- **nlohmann/json**: MIT License
- **zlib**: zlib License
- **ChaCha8**: Public domain (implementation in code)

No GPL dependencies = compatible with commercial use.

## Conclusion

The KX_SaveSystem provides a production-ready, efficient, and maintainable save system for UPBGE. Its pure C++ core ensures compatibility with Python-free builds (essential for Android), while optional Python bindings provide a convenient scripting interface.

Key advantages:
- ✅ Performance: MessagePack serialization, zlib compression
- ✅ Portability: Works with/without Python
- ✅ Robustness: Atomic operations, checksums, encryption
- ✅ Maintainability: Clean C++ architecture, minimal Blender impact
- ✅ Flexibility: Multiple formats, partial saves, version migration

The system is ready for integration into UPBGE 0.50 and future Android ports.
