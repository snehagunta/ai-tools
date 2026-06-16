---
name: breaking-changes
description: Detect breaking changes across API, database, config, behavior, and integrations
args: "all | api | database | config | behavior | integration"
context: fork
agent: Explore
allowed-tools: Bash(git *) Read Glob Grep
---

# Breaking Change Detection for Kessel Inventory API

Analyze changes for breaking changes across multiple dimensions, enforcing domain guidelines from AGENTS.md.

## Scope Selection

**Args parameter**: `{{args}}`

Run specific checks based on args:
- `all` or empty: Run all checks (default)
- `api`: Only API/protobuf breaking changes
- `database`: Only database schema breaking changes
- `config`: Only configuration breaking changes
- `behavior`: Only behavior breaking changes
- `integration`: Only integration breaking changes

## Current Changes

First, get the current diff to analyze:

!`git diff HEAD --no-color 2>/dev/null || echo "No changes in working directory"`

If no changes in working directory, check the current branch against main:

!`git diff main...HEAD --no-color 2>/dev/null || echo "No branch diff available"`

Also list modified files for context:

!`git status --short 2>/dev/null || echo "Not in git repository"`

---

## Check 1: API Breaking Changes (Protobuf/gRPC)

**Run this check if**: args is "all", "api", or empty

**Reference**: [API Contracts Guidelines](../../../docs/api-contracts-guidelines.md)

### What to Check

Examine changes to `.proto` files for:

1. **Field removal/renaming** (BREAKING for gRPC)
   - Removed fields without proper deprecation
   - Renamed fields without field number preservation
   - Changed field numbers (breaks wire format)

2. **Type changes** (BREAKING)
   - Changed field types (string → int, etc.)
   - Changed message types in fields
   - Changed repeated/optional/required modifiers

3. **Service endpoint changes** (BREAKING)
   - Removed RPC methods
   - Changed RPC method signatures
   - Changed request/response message types

4. **Enum changes** (BREAKING)
   - Removed enum values
   - Changed enum value numbers
   - Renamed enums without aliases

5. **Message nesting changes** (BREAKING)
   - Moved nested messages
   - Changed message hierarchy

### How to Check

!`git diff HEAD --no-color -- '*.proto' | head -300`

If no diff in working directory, check branch diff:

!`git diff main...HEAD --no-color -- '*.proto' | head -300`

Also check if buf.build detected breaking changes:

!`make api 2>&1 | grep -i "breaking" || echo "No buf breaking change warnings"`

### Analysis

For each breaking change found:
- **File**: `path/to/file.proto:line`
- **Change**: What was changed
- **Impact**: Why it's breaking
- **Severity**: HIGH/MEDIUM/LOW
- **Remediation**: How to fix (deprecation, field aliases, version bump)
- **Migration**: What consumers need to do

### Guidelines Reference

From [API Contracts Guidelines](../../../docs/api-contracts-guidelines.md):
- Use buf.build for breaking change detection
- Follow semantic versioning for APIs
- Deprecate before removing
- Use field aliases for renames
- Never reuse field numbers

---

## Check 2: Database Schema Breaking Changes

**Run this check if**: args is "all", "database", or empty

**Reference**: [Database Guidelines](../../../docs/database-guidelines.md)

### What to Check

Examine migration files for:

1. **Column removal** (BREAKING)
   - Dropped columns without deprecation period
   - Removed columns still referenced in code

2. **Type changes** (POTENTIALLY BREAKING)
   - Type changes that lose precision (int64 → int32)
   - Type changes that lose data (text → varchar(50))
   - Type changes incompatible with existing data

3. **Constraint additions** (POTENTIALLY BREAKING)
   - NOT NULL on existing columns without defaults
   - UNIQUE constraints on non-unique data
   - CHECK constraints that existing data violates
   - Foreign key constraints on invalid references

4. **Index removal** (PERFORMANCE BREAKING)
   - Removed indexes that queries depend on
   - Changed composite index column order

5. **Migration safety** (OPERATIONAL)
   - Missing advisory locks (required per guidelines)
   - Missing transaction wrapping
   - Long-running operations without batching

### How to Check

!`git diff HEAD --no-color -- 'internal/data/migrations/*.go' '*.sql' | head -300`

If no diff in working directory:

!`git diff main...HEAD --no-color -- 'internal/data/migrations/*.go' '*.sql' | head -300`

Check for new migration files:

!`git diff main...HEAD --name-only -- 'internal/data/migrations/*.go' | grep -v "^$"`

### Analysis

For each breaking change:
- **File**: Migration file path
- **Change**: What schema change is made
- **Impact**: Data loss, constraint violation, performance degradation
- **Severity**: HIGH (data loss), MEDIUM (constraints), LOW (performance)
- **Remediation**: Add deprecation period, backfill data, add defaults
- **Migration**: How to safely roll out

### Guidelines Reference

From [Database Guidelines](../../../docs/database-guidelines.md):
- All migrations MUST use advisory locks
- Use serializable transaction isolation
- Add columns with defaults before NOT NULL
- Drop columns in separate migration after deprecation
- Test migrations against production-like data volume

---

## Check 3: Configuration Breaking Changes

**Run this check if**: args is "all", "config", or empty

**Reference**: Project README and docker-compose documentation

### What to Check

Examine config changes for:

1. **Removed config keys** (BREAKING)
   - Deleted keys without deprecation
   - Removed environment variable support

2. **Renamed config keys** (BREAKING)
   - Changed key names without aliases
   - Changed nested structure

3. **Type changes** (BREAKING)
   - Changed value types (string → int, bool → string)
   - Changed structure (scalar → object, object → array)

4. **Default value changes** (POTENTIALLY BREAKING)
   - Changed defaults that affect behavior
   - Removed default values (now required)

5. **Viper env var compatibility** (BREAKING)
   - Keys with hyphens can't be overridden via env vars
   - Only dot-separated keys work with `INVENTORY_API_*` overrides

6. **Docker compose sync** (OPERATIONAL)
   - Config files in `development/configs/*.yaml` outdated
   - Environment variable overrides in `docker-compose.yaml` stale
   - Makefile targets referencing old config keys

### How to Check

!`git diff HEAD --no-color -- 'cmd/root.go' 'internal/config/*.go' 'development/configs/*.yaml' 'development/*.yaml' | head -300`

If no diff:

!`git diff main...HEAD --no-color -- 'cmd/root.go' 'internal/config/*.go' 'development/configs/*.yaml' 'development/*.yaml' | head -300`

Check for CLI flag changes:

!`git diff HEAD --no-color -- 'cmd/*.go' | grep -E "(Flag|Env|Default)" | head -50`

### Analysis

For each breaking change:
- **File**: Config file or code path
- **Change**: What config key/default changed
- **Impact**: Services that rely on old config
- **Severity**: HIGH (removed required), MEDIUM (renamed), LOW (default change)
- **Remediation**: Add deprecation warnings, support both old/new keys
- **Migration**: Update config files, env vars, compose files

### Special Check: Docker Compose Sync

Per AGENTS.md guidelines, verify:
- [ ] `development/configs/*.yaml` reflects new config keys
- [ ] `docker-compose.yaml` env vars match new structure
- [ ] Makefile targets pass correct config names
- [ ] `docs/dev-guides/docker-compose-options.md` updated

---

## Check 4: Behavior Breaking Changes

**Run this check if**: args is "all", "behavior", or empty

**Reference**: All domain guidelines in docs/

### What to Check

Examine code changes for behavioral breaks:

1. **Function signature changes** (BREAKING for public APIs)
   - Added required parameters
   - Removed parameters
   - Changed parameter types
   - Changed return types

2. **Error type changes** (BREAKING)
   - Changed error values/codes
   - Changed error hierarchy
   - Changed when errors are returned

3. **Authentication/Authorization changes** (BREAKING)
   - Changed auth requirements (public → authenticated)
   - Changed permission requirements
   - Changed identity extraction logic

4. **Timing/retry behavior** (BREAKING)
   - Changed timeout defaults
   - Changed retry logic
   - Changed backoff strategies

5. **Data format changes** (BREAKING)
   - Changed serialization format
   - Changed validation rules (stricter)
   - Changed normalization logic

6. **Domain type changes** (BREAKING)
   - Changed tiny type constructors (New* functions)
   - Changed validation in constructors
   - Changed normalization in constructors

### How to Check

!`git diff HEAD --no-color -- 'internal/biz/*.go' 'internal/service/*.go' | head -400`

If no diff:

!`git diff main...HEAD --no-color -- 'internal/biz/*.go' 'internal/service/*.go' | head -400`

Check for signature changes in public interfaces:

!`git diff HEAD --no-color -- 'internal/biz/model/*.go' | grep -E "(^-func|^+func|^-type|^+type)" | head -100`

### Analysis

For each breaking change:
- **File**: Code file path with line number
- **Change**: What behavior changed
- **Impact**: Callers that depend on old behavior
- **Severity**: HIGH (auth), MEDIUM (errors), LOW (timeouts)
- **Remediation**: Versioned APIs, feature flags, gradual rollout
- **Migration**: Code changes consumers need

### Guidelines Reference

From domain guidelines:
- [Security Guidelines](../../../docs/security-guidelines.md): Never bypass auth
- [Error Handling Guidelines](../../../docs/error-handling-guidelines.md): Maintain error codes
- [Performance Guidelines](../../../docs/performance-guidelines.md): Timeout consistency

### Special Check: Domain Model Changes

Per AGENTS.md, verify domain type changes:
- [ ] All `New*` constructors still validate invariants
- [ ] No direct type conversions bypassing constructors
- [ ] `Deserialize*` functions only used from storage/wire
- [ ] No protobuf types leaked into model layer

---

## Check 5: Integration Breaking Changes

**Run this check if**: args is "all", "integration", or empty

**Reference**: [Integration Guidelines](../../../docs/integration-guidelines.md)

### What to Check

Examine integration point changes:

1. **Kafka event schema changes** (BREAKING)
   - Changed event field names/types
   - Changed event structure
   - Removed event fields
   - Changed topic names

2. **Relations API contract changes** (BREAKING)
   - Changed relationship tuple format
   - Changed query patterns
   - Changed permission checks

3. **Health check response changes** (BREAKING for orchestrators)
   - Changed health check endpoint
   - Changed response format
   - Changed readiness/liveness criteria

4. **Service dependency version changes** (POTENTIALLY BREAKING)
   - Major version bumps in dependencies
   - Changed minimum version requirements
   - Incompatible protocol versions

5. **Outbox pattern changes** (BREAKING)
   - Changed WAL/table mode behavior
   - Changed event publishing guarantees
   - Changed deduplication logic

### How to Check

!`git diff HEAD --no-color -- 'internal/eventing/*.go' 'internal/relations/*.go' 'go.mod' | head -300`

If no diff:

!`git diff main...HEAD --no-color -- 'internal/eventing/*.go' 'internal/relations/*.go' 'go.mod' | head -300`

Check Kafka consumer changes:

!`git diff HEAD --no-color -- 'internal/service/consumer/*.go' | head -200`

Check go.mod for dependency bumps:

!`git diff HEAD --no-color -- 'go.mod' | grep -E "^[-+]" | grep -v "//" | head -50`

### Analysis

For each breaking change:
- **File**: Integration code path
- **Change**: What contract changed
- **Impact**: External systems that depend on contract
- **Severity**: HIGH (schema), MEDIUM (health), LOW (versions)
- **Remediation**: Dual-write period, version negotiation, gradual rollout
- **Migration**: Coordination needed with consuming services

### Guidelines Reference

From [Integration Guidelines](../../../docs/integration-guidelines.md):
- Kafka: Use Debezium CDC format, maintain schema compatibility
- Relations API: Follow Kessel relations service contract
- Health checks: Maintain /livez and /readyz semantics
- Observability: Maintain metrics and log structure

---

## Final Report

### Summary

Aggregate findings across all enabled checks:

**Breaking Changes Found**: [total count]

By category:
- API Breaking Changes: [count] (HIGH: X, MEDIUM: Y, LOW: Z)
- Database Breaking Changes: [count] (HIGH: X, MEDIUM: Y, LOW: Z)
- Configuration Breaking Changes: [count] (HIGH: X, MEDIUM: Y, LOW: Z)
- Behavior Breaking Changes: [count] (HIGH: X, MEDIUM: Y, LOW: Z)
- Integration Breaking Changes: [count] (HIGH: X, MEDIUM: Y, LOW: Z)

### Risk Assessment

**Overall Risk Level**: HIGH | MEDIUM | LOW

- **HIGH**: Data loss, auth bypass, incompatible schema changes
- **MEDIUM**: Behavior changes requiring code updates, config migrations
- **LOW**: Performance changes, default value changes, deprecations

### Recommendations

For each HIGH severity finding:
1. **Immediate action required** before merge
2. **Remediation steps** with code examples
3. **Migration plan** for affected systems

For MEDIUM findings:
1. **Coordinate with consumers** before merge
2. **Document migration** in release notes
3. **Provide backward compatibility** where possible

For LOW findings:
1. **Document in changelog**
2. **Notify in release announcement**
3. **Provide upgrade guide**

### Blockers

If any HIGH severity breaking changes found without remediation:
- **DO NOT COMMIT** until fixed or explicitly acknowledged
- **DO NOT PUSH** to shared branches
- **COORDINATE** with affected teams before proceeding

### Next Steps

1. Review each finding with file:line references
2. Apply remediation strategies
3. Update documentation (CHANGELOG, migration guides)
4. Re-run `/breaking-changes` to verify fixes
5. Coordinate with consuming services if needed

---

## Usage Examples

```bash
# Run all checks
/breaking-changes

# Check only API changes
/breaking-changes api

# Check only database migrations
/breaking-changes database

# Check config and behavior
/breaking-changes config
/breaking-changes behavior

# Check integrations
/breaking-changes integration
```

## Integration with PR Workflow

This skill can be integrated into your 13-stage PR workflow:

**Stage 12a: Breaking Change Detection** (before Code Review)
1. Run `/breaking-changes` on current diff
2. Review findings and apply remediations
3. Document breaking changes in PLAN.md
4. Only proceed to Code Review if no HIGH severity blockers

Add to `.pr-workflow-prompts.md`:
```
Before requesting code review, run:
/breaking-changes

If HIGH severity findings, fix before proceeding.
If MEDIUM findings, document migration plan.
```
