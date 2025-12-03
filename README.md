# Phase 4 Combined Integration Test: Mixed Poetry/UV Workspace

## Test ID
T-P4-006

## Category
Combined Integration - Workspace with Mixed Package Managers

## Priority
P1

## Description
This fixture simulates a workspace/monorepo during migration from Poetry to UV, where different packages are at different stages of migration. Some packages still use Poetry, some are fully migrated to UV, and some have both configurations.

## Features Combined
1. **UV Workspace** - Root-level workspace configuration
2. **Mixed Package Managers** - Different packages use different tools
3. **Internal Dependencies** - Packages depend on each other
4. **Migration Stages** - Packages at different migration stages
5. **Environment Variables** - Control which package manager to prefer per package
6. **Dual Lock Files** - Some packages have both poetry.lock and uv.lock

## Real-World Inspiration
Based on patterns from:
- Large monorepos during Poetry→UV migration
- Enterprise codebases with gradual migrations
- Teams transitioning incrementally
- Projects maintaining backward compatibility

## Workspace Structure
```
mixed-poetry-uv-workspace/
├── pyproject.toml (UV workspace root)
├── packages/
│   ├── legacy-service/
│   │   ├── pyproject.toml (Poetry-only)
│   │   └── poetry.lock
│   ├── new-service/
│   │   ├── pyproject.toml (UV-only, PEP 621)
│   │   └── uv.lock
│   └── shared-lib/
│       ├── pyproject.toml (Both Poetry and UV)
│       ├── poetry.lock
│       └── uv.lock
└── uv.lock (workspace-level lock, if UV manages workspace)
```

## Package Details

### Package 1: legacy-service (Poetry-Only)
- **Status**: Not yet migrated
- **Configuration**: [tool.poetry] only
- **Lock File**: poetry.lock only
- **Dependencies**: Depends on shared-lib
- **Challenge**: How to integrate Poetry package in UV workspace

### Package 2: new-service (UV-Only)
- **Status**: Fully migrated
- **Configuration**: [project] + [tool.uv] (PEP 621)
- **Lock File**: uv.lock only
- **Dependencies**: Depends on shared-lib
- **Challenge**: Standard UV package

### Package 3: shared-lib (Mixed)
- **Status**: During migration
- **Configuration**: Both [tool.poetry] and [project]
- **Lock Files**: Both poetry.lock and uv.lock
- **Dependencies**: Base library used by others
- **Challenge**: Which lock file to use for dependents

## Test Objectives
1. Verify workspace with mixed package managers detected
2. Verify per-package package manager identification
3. Verify internal dependencies across different managers
4. Verify workspace-level vs package-level lock files
5. Verify environment variable control per package
6. Verify migration path recommendations

## Success Criteria
- [ ] All 3 workspace members detected
- [ ] Package manager identified per package (Poetry/UV/Mixed)
- [ ] Internal dependencies resolved (services→shared-lib)
- [ ] Poetry-only package handled in UV workspace
- [ ] UV-only package handled correctly
- [ ] Mixed package uses correct lock file
- [ ] Workspace-level lock file used when available
- [ ] Clear warnings for mixed configurations
- [ ] Migration guidance provided

## Expected Behavior

### Scenario 1: UV Workspace with Mixed Members
```bash
# Scan workspace (default: prefer UV)
dependency-tree-builder scan .

# Expected output:
# - new-service: Uses uv.lock (UV-native)
# - shared-lib: Uses uv.lock (UV preferred)
# - legacy-service: Uses poetry.lock (Poetry-only) with warning
```

### Scenario 2: Per-Package Manager Selection
```bash
# Force Poetry for shared-lib
SHARED_LIB_PREFER_POETRY=true dependency-tree-builder scan .

# Ignore Poetry entirely (UV-only workspace)
IGNORE_POETRY=true dependency-tree-builder scan .
# Expected: Error or warning for legacy-service
```

### Scenario 3: Dependency Resolution
When new-service depends on shared-lib:
- Check: Which version of shared-lib? (poetry.lock or uv.lock)
- Expected: Use uv.lock for consistency unless PREFER_POETRY set

## Migration Patterns

### Pattern 1: Bottom-Up Migration
1. Start: All packages use Poetry
2. Migrate: Shared library first (creates both lock files)
3. Migrate: Services one by one
4. End: Remove poetry.lock files, UV-only

### Pattern 2: Top-Down Migration
1. Start: All packages use Poetry
2. Add: UV workspace root
3. Migrate: New services to UV, keep legacy on Poetry
4. Eventually: Migrate or replace legacy services

### Pattern 3: Side-by-Side
1. Keep: Poetry configuration for compatibility
2. Add: UV configuration as primary
3. Maintain: Both lock files
4. Tools: Support both package managers

## UV Version Compatibility
- Minimum: UV 0.3.0+ (workspace support)
- Recommended: UV 0.7.0+

## Files in this Fixture
- `pyproject.toml` - Workspace root (UV)
- `packages/legacy-service/pyproject.toml` - Poetry config
- `packages/legacy-service/poetry.lock` - Poetry lock
- `packages/new-service/pyproject.toml` - UV config
- `packages/new-service/uv.lock` - UV lock
- `packages/shared-lib/pyproject.toml` - Mixed config
- `packages/shared-lib/poetry.lock` - Poetry lock
- `packages/shared-lib/uv.lock` - UV lock
- `uv.lock` - Workspace lock (optional)
- `README.md` - This file

## Usage
```bash
# Initialize UV workspace (if not done)
uv sync

# Scan workspace
dependency-tree-builder scan . > workspace_output.json

# Scan individual package
dependency-tree-builder scan packages/new-service/ > new_service_output.json

# Force Poetry for specific package
SHARED_LIB_PREFER_POETRY=true dependency-tree-builder scan .
```

## Mend Integration Notes
This fixture tests Mend's ability to:
- Detect workspace structure
- Handle mixed package managers in workspace
- Scan each workspace member correctly
- Identify which package manager per member
- Report vulnerabilities across mixed workspace
- Provide unified security view

**Mend Configuration for Mixed Workspace:**
```properties
# Mend should detect workspace structure
python.resolveHierarchyTree=true

# For Poetry packages
python.poetryPath=poetry

# For UV packages
python.packageManager=uv

# Workspace-level scan
python.scanWorkspaces=true
```

## Related Tests
- T-P4-002: Workspace Monorepo
- T-P4-005: Mixed Poetry/UV (single project)
- T-P2-011: Workspace Monorepo (UV-only)

## Edge Cases

### Edge Case 1: Circular Dependencies Between Mixed Managers
- Package A (Poetry) depends on Package B (UV)
- Package B (UV) depends on Package A (Poetry)
- Expected: Detect circular dependency, warn about mixed managers

### Edge Case 2: Version Conflicts
- legacy-service depends on shared-lib (via poetry.lock: v1.0.0)
- new-service depends on shared-lib (via uv.lock: v1.1.0)
- Expected: Flag version mismatch across workspace

### Edge Case 3: Poetry Workspace vs UV Workspace
- Root has [tool.poetry] workspace
- But also has [tool.uv.workspace]
- Expected: Prefer UV workspace, warn about Poetry workspace

## Test Variations
1. **All Packages Poetry** - Baseline, all Poetry
2. **Mixed Migration** - This fixture
3. **All Packages UV** - End state, all migrated
4. **Poetry Workspace + UV Packages** - Reverse scenario
5. **No Workspace Config** - Detect as multi-project repo