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

---

## Test T-P1-006: pyproject.toml Parsing

**Category:** Manifest Parsing
**Priority:** P0
**UV Version:** All (0.1.0+)
**Complexity:** Medium

**Feature Dependencies:**
- PEP 621 project metadata (see `research/uv_features.md#pep-621-support`)
- TOML parsing
- UV-specific `[tool.uv]` section

**Inspired By:**
- Pattern: PEP 621 compliance (100% of analyzed projects - see `research/patterns.md`)
- GitHub Project: modelcontextprotocol/python-sdk (see `research/github_projects.md#project-9`)
- Real-World: Standard Python project configuration

**Real-World Relevance:** CRITICAL
- Found in 17/17 analyzed projects (100%)
- Primary source of truth for direct dependencies
- Required for both CLI and GitHub App scanning

### Objective

Verify that the dependency tree builder correctly parses pyproject.toml files, extracting project metadata and direct dependencies according to PEP 621 and UV-specific conventions.

### Setup

1. Create directory: `fixtures/phase_1/pyproject_toml_parsing/`
2. Create comprehensive `pyproject.toml` with all relevant sections
3. Run dependency tree builder (no uv.lock - test manifest parsing only)

### Input Files

**Location:** `fixtures/phase_1/pyproject_toml_parsing/`

**pyproject.toml:**
```toml
[project]
name = "test-pyproject-parsing"
version = "0.1.0"
description = "Comprehensive pyproject.toml parsing test"
readme = "README.md"
requires-python = ">=3.10"
license = {text = "MIT"}
authors = [
    {name = "Test Author", email = "test@example.com"}
]
keywords = ["testing", "uv", "dependencies"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Programming Language :: Python :: 3.10",
]

dependencies = [
    "fastapi>=0.109.0",
    "uvicorn[standard]>=0.27.0",
    "sqlalchemy>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.3.0",
]
docs = [
    "mkdocs>=1.5.0",
    "mkdocs-material>=9.5.0",
]

[project.urls]
Homepage = "https://example.com"
Documentation = "https://docs.example.com"
Repository = "https://github.com/example/repo"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv]
dev-dependencies = [
    "mypy>=1.8.0",
]

[tool.uv.sources]
# Private package index example
my-private-package = { index = "https://private.pypi.org/simple" }
```

### Expected Behavior

The dependency tree builder should:

1. **Extract Project Metadata:**
   - name, version, description
   - Python version requirement
   - License, authors, keywords
   - URLs (homepage, repository, etc.)

2. **Parse Dependencies:**
   - Runtime dependencies from `dependencies = [...]`
   - Optional dependencies from `[project.optional-dependencies]`
   - Dev dependencies from `[tool.uv]` section

3. **Handle Extras:**
   - Recognize extras syntax: `uvicorn[standard]`
   - Parse correctly even without lock file

4. **Identify UV-Specific Sections:**
   - `[tool.uv]` configuration
   - `[tool.uv.sources]` for custom indexes
   - `[tool.uv.dev-dependencies]` (older format)

**Expected Output Structure:**

```json
{
  "project": {
    "name": "test-pyproject-parsing",
    "version": "0.1.0",
    "description": "Comprehensive pyproject.toml parsing test",
    "python_version": ">=3.10",
    "license": "MIT",
    "authors": [
      {"name": "Test Author", "email": "test@example.com"}
    ],
    "urls": {
      "homepage": "https://example.com",
      "repository": "https://github.com/example/repo"
    }
  },
  "dependencies": {
    "runtime": {
      "direct": [
        {
          "name": "fastapi",
          "constraint": ">=0.109.0",
          "source": "pyproject.toml"
        },
        {
          "name": "uvicorn",
          "constraint": ">=0.27.0",
          "extras": ["standard"],
          "source": "pyproject.toml"
        },
        {
          "name": "sqlalchemy",
          "constraint": ">=2.0.0",
          "source": "pyproject.toml"
        }
      ]
    },
    "optional": {
      "dev": [
        {"name": "pytest", "constraint": ">=8.0.0"},
        {"name": "ruff", "constraint": ">=0.3.0"}
      ],
      "docs": [
        {"name": "mkdocs", "constraint": ">=1.5.0"},
        {"name": "mkdocs-material", "constraint": ">=9.5.0"}
      ]
    },
    "uv_dev": [
      {"name": "mypy", "constraint": ">=1.8.0"}
    ]
  },
  "uv_config": {
    "sources": {
      "my-private-package": {
        "index": "https://private.pypi.org/simple"
      }
    }
  }
}
```

### Success Criteria

- [x] All project metadata extracted correctly
- [x] Runtime dependencies parsed with constraints
- [x] Optional dependencies grouped correctly
- [x] UV-specific dev dependencies identified
- [x] Extras syntax parsed (uvicorn[standard])
- [x] Custom sources/indexes detected
- [x] Python version requirement extracted
- [x] No errors on valid pyproject.toml
- [x] Clear error messages on invalid TOML

### Version Notes

**All UV Versions:**
- PEP 621 support is consistent
- `[tool.uv]` section structure may vary slightly

### Edge Cases

1. **No Dependencies:**
   ```toml
   [project]
   name = "empty-project"
   dependencies = []
   ```
   - Should return empty dependency arrays

2. **Complex Extras:**
   ```toml
   dependencies = [
       "package[extra1,extra2,extra3]>=1.0.0"
   ]
   ```
   - Should parse all extras

3. **Invalid TOML:**
   - Missing closing bracket, invalid syntax
   - Should provide helpful error message

4. **Missing Required Fields:**
   ```toml
   [project]
   # Missing name or version
   ```
   - Should error with "Missing required field: name"

### Mend Integration

**Critical:** Mend needs pyproject.toml when uv.lock is unavailable or for validating lock file.

**Mend Configuration:**

For Mend to scan UV projects, pyproject.toml must be PEP 621 compliant:

```properties
# whitesource.config
python.resolveDependencies=true
```

**Validation:**
- Tool's pyproject.toml parsing should match what Mend extracts
- Mend may not recognize `[tool.uv]` sections (UV under development)
- Ensure PEP 621 `dependencies = [...]` section is complete

**See:** `research/mend_integration.md#pyproject-toml-detection`

### Automation

**Priority:** CRITICAL
**Framework:** pytest
**Fixture:** `fixtures/phase_1/pyproject_toml_parsing/`
**Test File:** `tests/test_phase_1_critical.py::test_pyproject_parsing`

---

## Test T-P1-007: Mend Baseline Integration

**Category:** Mend Integration
**Priority:** P0
**UV Version:** All (0.9.x)
**Complexity:** High

**Feature Dependencies:**
- Complete dependency tree resolution (all above tests)
- Mend CLI integration (see `research/mend_integration.md#mend-cli-tool`)
- Mend Unified Agent compatibility
- Requirements.txt export for Mend

**Inspired By:**
- Mend integration requirements (see `research/mend_integration.md`)
- Real-World: All 17 projects will be scanned with Mend
- Critical for production use

**Real-World Relevance:** CRITICAL
- 100% of projects will require Mend scanning
- Blocks production deployment if Mend integration fails
- Essential for security vulnerability tracking

### Objective

Verify that the dependency tree builder produces output compatible with Mend scanning, and that dependency trees from our tool match (or acceptably differ from) Mend's dependency resolution for UV projects.

### Setup

1. Create directory: `fixtures/phase_1/mend_baseline_integration/`
2. Create UV project with mix of direct and transitive dependencies
3. Generate uv.lock
4. Run dependency tree builder
5. Export to requirements.txt
6. Run Mend scan
7. Compare results

### Input Files

**Location:** `fixtures/phase_1/mend_baseline_integration/`

**pyproject.toml:**
```toml
[project]
name = "test-mend-integration"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.109.0",
    "sqlalchemy>=2.0.0",
    "requests>=2.31.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.3.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

**Mend Configuration Files:**

**.whitesource:**
```json
{
  "scanSettings": {
    "configMode": "LOCAL"
  },
  "checkRunSettings": {
    "vulnerableCheckRunConclusionLevel": "failure"
  }
}
```

**whitesource.config:**
```properties
python.resolveDependencies=true
python.installVirtualenv=true
python.resolveHierarchyTree=true
python.path=python3.11
python.pipenvDevDependencies=true
```

### Test Procedure

**Step 1: Generate UV Dependency Tree**
```bash
cd fixtures/phase_1/mend_baseline_integration/
uv lock
dependency-tree-builder scan . > uv_tree.json
```

**Step 2: Export for Mend**
```bash
uv pip compile pyproject.toml -o requirements.txt
uv pip compile pyproject.toml --group dev -o requirements-dev.txt
```

**Step 3: Run Mend CLI Scan**
```bash
mend dep --dir . --format json > mend_tree.json
```

**Step 4: Run Mend Unified Agent Scan**
```bash
java -jar wss-unified-agent.jar \
  -c whitesource.config \
  -d . \
  -wss.url https://saas.mend.io/agent \
  -generateScanReport true
```

**Step 5: Compare Results**
```bash
# Compare dependency counts
jq '.summary.total_dependencies' uv_tree.json
jq '.dependencies | length' mend_tree.json

# Compare direct dependencies
jq '.dependencies.direct[].name' uv_tree.json | sort
jq '.dependencies[] | select(.type=="DIRECT") | .name' mend_tree.json | sort

# Compare transitive dependencies
jq '.dependencies.transitive[].name' uv_tree.json | sort
jq '.dependencies[] | select(.type=="TRANSITIVE") | .name' mend_tree.json | sort
```

### Expected Behavior

**Dependency Count Comparison:**
- UV tool total count: ~25 dependencies
- Mend total count: ~25 dependencies (±2 acceptable)
- Discrepancies should be documented

**Direct Dependency Comparison:**
- Should match exactly: fastapi, sqlalchemy, requests
- Mend should detect all 3 direct dependencies

**Transitive Dependency Comparison:**
- Core transitive deps should match (pydantic, starlette, etc.)
- Minor version differences acceptable
- Document any packages detected by one but not the other

**Expected Output Comparison:**

```json
{
  "comparison": {
    "uv_tool": {
      "total_dependencies": 25,
      "direct_count": 3,
      "transitive_count": 22,
      "max_depth": 3
    },
    "mend_cli": {
      "total_dependencies": 24,
      "direct_count": 3,
      "transitive_count": 21,
      "max_depth": 3
    },
    "differences": {
      "missing_in_mend": [],
      "missing_in_uv_tool": ["some-build-dependency"],
      "version_mismatches": [
        {
          "package": "urllib3",
          "uv_version": "2.2.1",
          "mend_version": "2.2.0"
        }
      ]
    },
    "match_percentage": 96,
    "acceptable": true
  }
}
```

### Success Criteria

- [x] Mend CLI scan completes successfully
- [x] Mend Unified Agent scan completes successfully
- [x] Direct dependencies match 100%
- [x] Transitive dependencies match ≥95%
- [x] Total dependency count within ±5%
- [x] Version mismatches documented with explanation
- [x] No critical vulnerabilities missed by either tool
- [x] Mend dependency tree visualization shows correct hierarchy
- [x] Dev dependencies correctly separated in Mend results
- [x] Dependency types (runtime vs dev) match between tools

### Acceptable Discrepancies

**1. Build Dependencies:**
- Mend may detect setuptools, wheel, etc. not in uv.lock
- Acceptable if documented

**2. Version Micro Differences:**
- UV: urllib3 2.2.1
- Mend: urllib3 2.2.0
- Acceptable if both are within constraint range

**3. Platform-Specific Dependencies:**
- Mend may detect Windows-only or Linux-only packages
- Acceptable if explained by platform differences

**4. Optional Dependencies:**
- If extras are resolved differently
- Document which extras each tool included

### Version Notes

**Mend UV Support Status:**
- UV is under development in Mend
- Current workaround: requirements.txt export
- This test establishes baseline for when UV is natively supported

**See:** `research/mend_integration.md#uv-specific-considerations`

### Edge Cases

1. **Mend Fails to Resolve:**
   - If Mend cannot parse pyproject.toml without requirements.txt
   - Document error, confirm requirements.txt resolves issue

2. **Significant Discrepancies (>10% difference):**
   - Investigate root cause
   - May indicate bug in either tool
   - Document for Mend team

3. **Vulnerability Detection Differences:**
   - If Mend detects vulnerability not in uv.lock
   - May indicate dependency resolution difference
   - Critical to document

### Mend Integration Details

**Mend Platform Validation:**

After scan, verify in Mend web UI (https://app.mend.io/):

1. **Project Created:** "test-mend-integration" appears in Mend
2. **Dependency Tree:** Tree view shows correct hierarchy
3. **Vulnerability Report:** All vulnerabilities mapped to correct dependencies
4. **Direct vs Transitive:** Dependencies correctly categorized
5. **Dev Dependencies:** Separated and lower priority

**Mend Workflow Test:**

If vulnerabilities found, verify workflow triggers:
- High-severity vulnerability creates Jira ticket
- Transitive vulnerability shows parent dependency
- Fix recommendations reference correct direct dependency

**See:** `research/mend_integration.md#workflows-and-policies`

### Automation

**Priority:** CRITICAL
**Framework:** pytest + Mend CLI integration
**Fixture:** `fixtures/phase_1/mend_baseline_integration/`
**Test File:** `tests/test_phase_1_critical.py::test_mend_integration`
**External Dependencies:** Mend CLI, Mend API credentials

---

## Phase 1 Summary

### Test Overview

| Test ID | Name | Category | Complexity | Priority |
|---------|------|----------|------------|----------|
| T-P1-001 | Basic Dependency Tree Resolution | Core Dependencies | Simple | P0 |
| T-P1-002 | Development Dependencies Resolution | Dependency Types | Simple | P0 |
| T-P1-003 | Transitive Dependency Mapping | Dependency Tree Structure | Medium | P0 |
| T-P1-004 | Version Constraint Handling | Version Resolution | Medium | P0 |
| T-P1-005 | Lock File Format Compatibility | File Format Parsing | Medium | P0 |
| T-P1-006 | pyproject.toml Parsing | Manifest Parsing | Medium | P0 |
| T-P1-007 | Mend Baseline Integration | Mend Integration | High | P0 |

**Total Tests:** 7
**Estimated Implementation Time:** 12-16 hours
**Blocking Issues:** None (foundational tests)

### Coverage

**✓ Core Functionality:**
- Basic dependency resolution from uv.lock
- Development vs runtime dependency classification
- Transitive dependency mapping and hierarchy
- Version constraint parsing and validation
- Lock file compatibility across UV versions (0.1.x - 0.9.x)
- pyproject.toml manifest parsing (PEP 621)

**✓ Dependency Types:**
- Runtime dependencies (direct + transitive)
- Development dependencies (PEP 735 groups)
- Optional dependencies (extras)
- Shared dependencies (used by multiple parents)

**✓ Version Handling:**
- PEP 440 version specifiers (all operators)
- Version constraint satisfaction validation
- Lock file format evolution across UV versions
- Breaking changes handling (v0.3.0, v0.5.0, v0.7.0)

**✓ Mend Integration:**
- Baseline dependency tree comparison
- Requirements.txt export for Mend scanning
- CLI and Unified Agent compatibility
- Dependency type classification (runtime vs dev)
- Vulnerability mapping validation

### Test Execution Order

**Recommended Execution Sequence:**

1. **T-P1-006** - pyproject.toml Parsing (prerequisite for all others)
2. **T-P1-005** - Lock File Format Compatibility (prerequisite for parsing)
3. **T-P1-001** - Basic Dependency Tree Resolution (foundation)
4. **T-P1-004** - Version Constraint Handling (builds on T-P1-001)
5. **T-P1-002** - Development Dependencies Resolution (builds on T-P1-001)
6. **T-P1-003** - Transitive Dependency Mapping (builds on T-P1-001)
7. **T-P1-007** - Mend Baseline Integration (requires all above)

**Dependencies:**
- T-P1-007 depends on: All other P1 tests
- T-P1-002, T-P1-003, T-P1-004 depend on: T-P1-001
- T-P1-001 depends on: T-P1-005, T-P1-006

### Exit Criteria

**Phase 1 Complete When:**

- [x] All 7 tests implemented and passing
- [x] All fixtures created with valid UV projects
- [x] Mend integration validated with ≥95% match rate
- [x] Cross-version compatibility verified (UV 0.1.x - 0.9.x)
- [x] Clear error messages for all failure cases
- [x] Documentation complete for each test
- [x] Automation implemented in pytest

### Next Steps

**After Phase 1 Completion:**

1. **Phase 2: Extended Coverage (P1)**
   - Optional dependencies
   - Workspace management
   - Complex scenarios
   - Additional Mend integration tests

2. **Phase 3: Advanced Scenarios (P2)**
   - Large dependency trees (1000+ packages)
   - Performance testing
   - Edge cases and error handling

3. **Phase 4: Real Project Integration (P1)**
   - Test against all 17 analyzed GitHub projects
   - Validate patterns discovered in research
   - Document project-specific edge cases

4. **Phase 5: Version-Specific Testing**
   - Detailed tests for each UV version range
   - Migration testing between versions
   - Breaking change validation

5. **Phase 6: Performance Testing**
   - Large-scale dependency tree performance
   - Mend scan performance comparison
   - Memory and CPU profiling

---

## References

### Research Documents

- **UV Features:** `research/uv_features.md`
- **Version Matrix:** `research/version_matrix.md`
- **GitHub Projects:** `research/github_projects.md`
- **Breaking Changes:** `research/breaking_changes.md`
- **Patterns:** `research/patterns.md`
- **Mend Integration:** `research/mend_integration.md`

### External Documentation

- **UV Documentation:** https://docs.astral.sh/uv/
- **PEP 621 (Project Metadata):** https://peps.python.org/pep-0621/
- **PEP 440 (Version Specifiers):** https://peps.python.org/pep-0440/
- **PEP 735 (Dependency Groups):** https://peps.python.org/pep-0735/
- **Mend Documentation:** https://docs.mend.io/
- **Mend Python Configuration:** https://docs.mend.io/wsk/configuring-the-unified-agent-for-python

### Test Fixtures

All test fixtures are located in: `fixtures/phase_1/`

Each test has its own subdirectory:
- `basic_dependency_tree_resolution/` - Basic dependencies
- `development_dependencies/` - Development dependencies
- `transitive_dependency_mapping/` - Transitive mapping
- `version_constraint_handling/` - Version constraints
- `lock_file_format_compatibility/` - Lock file compatibility (with version subdirs)
- `pyproject_toml_parsing/` - pyproject.toml parsing
- `mend_baseline_integration/` - Mend integration

---

**Document Version:** 1.0
**Last Updated:** 2025-11-28
**Status:** Ready for Implementation
