## Test T-P1-005: Lock File Format Compatibility

**Category:** File Format Parsing
**Priority:** P0
**UV Version:** All (0.1.0 - 0.9.x+)
**Complexity:** Medium

**Feature Dependencies:**
- Lock file format evolution (see `research/version_matrix.md#lock-file-format-evolution`)
- TOML parsing
- Cross-version compatibility

**Inspired By:**
- Breaking change v0.3.0: Lock file format v1 introduction (see `research/breaking_changes.md#v030`)
- Breaking change v0.7.0: Enhanced lock file metadata (see `research/breaking_changes.md#v070`)
- Real-World: Projects migrating across UV versions

**Real-World Relevance:** CRITICAL
- All 17 projects use some version of UV
- Version distribution: 6% v0.1-0.3, 41% v0.4-0.6, 53% v0.7+ (see `research/github_projects.md#version-distribution-analysis`)
- Tool must handle all versions in the wild

### Objective

Verify that the dependency tree builder can parse uv.lock files across all UV versions (0.1.x through 0.9.x+), handling format changes gracefully and extracting dependency information consistently.

### Setup

1. Create directory: `fixtures/phase_1/lock_file_format_compatibility/`
2. Create subdirectories for each UV version range:
   - `v01_to_v03/` - Lock files from UV 0.1.x - 0.3.x
   - `v04_to_v06/` - Lock files from UV 0.4.x - 0.6.x
   - `v07_plus/` - Lock files from UV 0.7.x+
3. Generate lock files using different UV versions
4. Test dependency tree builder on each

### Input Files

**Location:** `fixtures/phase_1/lock_file_format_compatibility/v01_to_v03/`

**uv.lock (UV 0.1.x - 0.3.x format):**
```toml
version = 1
requires-python = ">=3.8"

[[package]]
name = "requests"
version = "2.31.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi" },
    { name = "charset-normalizer" },
    { name = "idna" },
    { name = "urllib3" },
]

[[package]]
name = "certifi"
version = "2024.7.4"
source = { registry = "https://pypi.org/simple" }

# ... more packages
```

**Location:** `fixtures/phase_1/lock_file_format_compatibility/v04_to_v06/`

**uv.lock (UV 0.4.x - 0.6.x format):**
```toml
version = 1
requires-python = ">=3.8"

[[package]]
name = "requests"
version = "2.31.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi", specifier = ">=2024.0.0" },
    { name = "charset-normalizer", specifier = ">=3.0.0,<4.0.0" },
    { name = "idna", specifier = ">=3.0" },
    { name = "urllib3", specifier = ">=2.0.0,<3.0.0" },
]

# Added metadata
[[package]]
name = "certifi"
version = "2024.7.4"
source = { registry = "https://pypi.org/simple" }
requires-dist = []

# ... more packages
```

**Location:** `fixtures/phase_1/lock_file_format_compatibility/v07_plus/`

**uv.lock (UV 0.7.x+ format):**
```toml
version = 1
requires-python = ">=3.8"

[[package]]
name = "requests"
version = "2.31.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi", specifier = ">=2024.0.0" },
    { name = "charset-normalizer", specifier = ">=3.0.0,<4.0.0" },
    { name = "idna", specifier = ">=3.0" },
    { name = "urllib3", specifier = ">=2.0.0,<3.0.0" },
]
wheels = [
    { url = "https://...", hash = "sha256:..." },
]
sdist = { url = "https://...", hash = "sha256:..." }

# Enhanced metadata in v0.7+
[[package]]
name = "certifi"
version = "2024.7.4"
source = { registry = "https://pypi.org/simple" }
requires-dist = []
wheels = [
    { url = "https://...", hash = "sha256:...", size = 162960 },
]

# ... more packages with enhanced metadata
```

### Expected Behavior

The dependency tree builder should:

1. **Detect Lock File Version:**
   - Read `version = 1` field
   - Identify UV version based on metadata presence

2. **Parse Core Fields (All Versions):**
   - name
   - version
   - source
   - dependencies array

3. **Handle Optional Fields:**
   - v0.4+: specifier, requires-dist
   - v0.7+: wheels, sdist, hash, size

4. **Extract Consistent Data:**
   - Regardless of format version, extract same core dependency tree
   - Additional metadata (wheels, hashes) is optional

**Expected Output Structure (Consistent Across Versions):**

```json
{
  "lock_file": {
    "format_version": 1,
    "uv_version_detected": "0.7.x+",
    "features_detected": ["wheels", "sdist", "hashes"]
  },
  "dependencies": {
    "direct": [
      {
        "name": "requests",
        "version": "2.31.0",
        "source": "https://pypi.org/simple",
        "dependencies": ["certifi", "charset-normalizer", "idna", "urllib3"]
      }
    ],
    "transitive": [
      {
        "name": "certifi",
        "version": "2024.7.4",
        "source": "https://pypi.org/simple"
      }
    ]
  }
}
```

### Success Criteria

- [x] Parses lock files from UV 0.1.x - 0.3.x
- [x] Parses lock files from UV 0.4.x - 0.6.x
- [x] Parses lock files from UV 0.7.x+
- [x] Extracts consistent dependency tree regardless of format version
- [x] Handles optional fields gracefully (no errors if missing)
- [x] Detects lock file format version
- [x] Documents which UV version range produced the lock file
- [x] No parsing errors across any version
- [x] Dependency relationships identical across formats

### Version Notes

**UV 0.1.x - 0.3.x:**
- Basic lock file format
- Minimal metadata

**UV 0.4.x - 0.6.x:**
- Added `specifier` field to dependencies
- Added `requires-dist` metadata

**UV 0.7.x+:**
- Added `wheels` and `sdist` distribution info
- Added `hash` and `size` for integrity verification
- Enhanced metadata throughout

**See:** `research/version_matrix.md#lock-file-format-evolution` for complete format history.

### Edge Cases

1. **Missing Optional Fields:**
   - v0.1-0.3 lock file has no `wheels` field
   - Parser should not error, simply skip optional fields

2. **Forward Compatibility:**
   - Future UV versions may add new fields
   - Parser should ignore unknown fields gracefully

3. **Malformed Lock Files:**
   - Invalid TOML syntax
   - Should provide clear error message

### Mend Integration

**Compatibility Note:** Mend does not parse uv.lock directly (UV under development in Mend). This test ensures our tool handles all UV versions, which Mend will need when UV support is complete.

**Validation:**
- Demonstrate consistent dependency extraction across UV versions
- Provide evidence that tool is future-proof for Mend UV integration
- Document format differences for Mend team

**See:** `research/mend_integration.md#uv-specific-considerations`

### Automation

**Priority:** CRITICAL
**Framework:** pytest
**Fixture:** `fixtures/phase_1/lock_file_format_compatibility/` (with subdirectories)
**Test File:** `tests/test_phase_1_critical.py::test_lock_file_compatibility`

