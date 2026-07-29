# Code Audit Methodology

This document provides a comprehensive, systematic approach to code quality auditing. Follow these phases for thorough analysis.

## Phase 0: Pre-Analysis Setup

Before analyzing code, establish the context:

### 1. Project Configuration
- **Package files**: package.json, requirements.txt, go.mod, pom.xml, etc.
- **Tech stack**: Identify languages, frameworks, and core libraries
- **Linting configs**: eslint, prettier, black, golint, etc.
- **Project docs**: CLAUDE.md, README.md for project-specific guidelines

### 2. Baseline Checks
Run existing linting and testing:
```bash
# Python (primary for jobs in this repo)
ruff check .
mypy .
pytest

# Other languages, if present in the repo
go vet ./... ; golint ./...
```

Document existing errors/warnings as baseline.

### 3. Documentation Loading
Use Context7 to pre-load documentation for identified core libraries:
```
mcp__context7__resolve-library-id  → Get library ID
mcp__context7__query-docs    → Load current best practices
```

## Phase 1: Discovery

### File Identification
Find all code files by type:
```
*.py                       (Python — primary for jobs in this repo)
*.js, *.ts                 (JavaScript/TypeScript)
*.go                       (Go)
*.rs                       (Rust)
*.rb                       (Ruby)
```

### Organization
- Group files by module/feature for contextual analysis
- Create a tracking list for systematic progress
- Prioritize core business logic over utilities

## Phase 2: File-by-File Analysis

For each file, analyze for the following categories:

### Dead Code
- Unused functions and methods
- Unused variables and imports
- Unreachable code blocks
- Commented-out code
- Deprecated features still present

### Code Smells & Anti-Patterns
- Functions longer than 50 lines
- High cyclomatic complexity (> 10)
- Deeply nested conditionals (> 3 levels)
- Magic numbers without constants
- Copy-paste code duplication
- God objects/functions doing too much
- Long parameter lists (> 5 params)

### Security Vulnerabilities
- Hardcoded secrets, API keys, passwords
- SQL injection vulnerabilities
- XSS (Cross-Site Scripting) risks
- Command injection risks
- Insecure deserialization
- Missing input validation
- Information disclosure in errors

### Performance Issues
- O(n²) or worse algorithms in hot paths
- Missing database indexes
- N+1 query patterns
- Unnecessary synchronous operations
- Missing caching for expensive operations
- Large memory allocations in loops
- Blocking I/O in async contexts

### TypeScript/Type Safety Issues
- Missing type annotations
- Excessive use of `any` type
- Type assertions that could be avoided
- Custom types duplicating official @types/* packages
- Missing null/undefined checks

### Async/Promise Issues
- Missing `await` keywords
- Unhandled promise rejections
- Callback hell that should use async/await
- Fire-and-forget promises without error handling

### Memory Leaks
- Event listeners not removed on cleanup
- Timers (setInterval, setTimeout) not cleared
- Large objects retained unnecessarily
- Closures holding references too long
- DOM references kept after element removal

### Error Handling
- Empty catch blocks
- Catch-and-ignore patterns
- Missing try/catch in async code
- Inconsistent error types
- Generic error messages hiding root cause

## Phase 3: Best Practices Verification

### Context7 Documentation Check
For every major library identified:

1. **Resolve library ID**:
   ```
   mcp__context7__resolve-library-id: "pydantic"
   ```

2. **Get current best practices**:
   ```
   mcp__context7__query-docs: {
     "context7CompatibleLibraryID": "/pydantic/pydantic",
     "topic": "v2 model validation best practices"
   }
   ```

3. **Focus areas**:
   - Migration guides between versions
   - Deprecated features and replacements
   - Performance best practices
   - Security considerations
   - Common pitfalls and anti-patterns

### GitHub Research
Use `gh` CLI to research real-world usage:

```bash
# Search for patterns
gh search code "signal.signal SIGTERM" --language=python

# Check repository health
gh repo view [library] --json stargazersCount,updatedAt,openIssues

# Look for security advisories
gh api /repos/{owner}/{repo}/security-advisories
```

### Cross-Reference Findings
- Compare actual implementation vs official documentation
- Identify deviations from recommended patterns
- Note outdated patterns that need modernization
- Flag anti-patterns explicitly discouraged in docs

## Phase 3.5: Type-Hint Verification (Python)

Perform additional type analysis on the codebase:

### Run the type checker
```bash
mypy .                 # strict where practical (see docs/job-standards.md)
```
Flag every `# type: ignore` without a justification, and any use of `Any` in the pure decision
core (allowed only at explicit boundary-parsing points).

### Check for redundant / hand-rolled types
Search for custom definitions that duplicate what the stdlib or a dependency already provides:
- Re-declaring a shape a `dataclass`/Pydantic model already defines elsewhere
- Custom "record" dicts (`dict[str, Any]`) where a typed dataclass exists for the same data
- Hand-written protocols that mirror `typing`/`collections.abc` ABCs

### Common Issues
- Untyped function signatures (every signature MUST be fully typed)
- `Any` leaking through the decision core instead of a concrete dataclass
- A state record persisted/loaded as a raw `dict` instead of its dataclass

## Phase 4: Pattern Detection

Look for recurring issues across the codebase:

### Cross-File Patterns
- Same anti-pattern repeated in multiple files
- Duplicated utility functions
- Inconsistent error handling approaches
- Different coding styles in different modules

### Abstraction Opportunities
- Repeated code that could be a shared utility function
- Common patterns that could be a decorator, context manager, or mixin
- Cross-cutting concerns (logging, retries, clamps) that belong in the shared command base

### Inconsistencies
- Mixed async styles (callbacks, promises, async/await)
- Inconsistent naming conventions
- Different error handling strategies
- Varying code organization patterns

## Phase 5: Library Recommendations

For custom implementations, find mature replacements:

### Discovery Process
1. **Check existing libraries first** - Use Context7 to see if current libraries already provide needed functionality
2. **Search package registries** - PyPI (and the stdlib first — prefer it before adding a dependency)
3. **Verify library health**:
   - Recent commits (active development)
   - Open issues (responsiveness)
   - Download stats (community adoption)
   - Security advisories (vulnerability history)

### Evaluation Criteria
- **Maintenance**: Last commit < 6 months
- **Adoption**: Significant download/star count
- **Security**: No unaddressed vulnerabilities
- **Import/startup cost**: heavy transitive dependencies slow a job's cold start; prefer the stdlib
- **API stability**: Semantic versioning, migration guides
- **Documentation**: Clear examples and API docs

### Common Replacements
| Custom Implementation | Recommended Library |
|----------------------|---------------------|
| Config validation | pydantic v2 |
| HTTP client | httpx, requests |
| Date/time handling | stdlib `datetime`/`zoneinfo` (arrow/pendulum only if justified) |
| Retry/backoff logic | tenacity |
| CLI parsing | stdlib `argparse` |
| YAML loading | PyYAML (`yaml.safe_load`) |
| Structured records | stdlib `dataclasses` |

## Phase 6: Report Generation

### Report Structure

#### Executive Summary (2-3 paragraphs)
- Total files analyzed
- High-level findings overview
- Key risks and recommendations

#### Critical Issues (Immediate Action)
For each:
- File path and line number
- Issue description
- Security/stability impact
- Fix example
- Effort estimate

#### High Priority Issues
- Performance bottlenecks
- Maintainability problems
- Missing error handling

#### Medium Priority Issues
- Best practices violations
- Code quality concerns
- Type safety improvements

#### Low Priority Issues
- Style inconsistencies
- Minor improvements
- Documentation gaps

#### Library Recommendations
For each suggested replacement:
- Current custom code location
- Recommended library
- Migration effort
- Added dependency weight (import/startup cost)

#### Quick Wins
Low-effort, high-value fixes:
- < 30 minutes to implement
- High impact on quality/security

#### Action Plan
Prioritized steps with:
- Effort estimates (S/M/L/XL)
- Dependencies between tasks
- Suggested sprint allocation

### Report Format Requirements

Each issue should include:
```markdown
### [PRIORITY] Issue Title
**Location:** `<job_package>/strategies/dca_grid/decision.py:42`

**Problem:**
Description of the issue and why it matters.

**Before:**
```python
# problematic code
```

**After:**
```python
# fixed code
```

**Effort:** S (< 30 min) | M (1-4 hours) | L (4-8 hours) | XL (> 8 hours)
```

## Tool Usage Reference

### Context7
```
# Resolve library ID first
mcp__context7__resolve-library-id: "ccxt"

# Then get documentation
mcp__context7__query-docs: {
  "context7CompatibleLibraryID": "/ccxt/ccxt",
  "topic": "create_order market"
}
```

### GitHub CLI
```bash
# Repository health
gh repo view owner/repo --json stargazersCount,updatedAt

# Code search
gh search code "pattern" --language=python

# Issues search
gh search issues "memory leak" --repo=owner/repo
```

### Package Research
Use `mcp__fetch__fetch` for package registry pages:
- pypi.org/project/[name]

## Common Pitfalls to Avoid

1. **Don't rely on assumptions** - Always verify with documentation
2. **Don't suggest outdated patterns** - Check current best practices
3. **Don't recommend unmaintained libraries** - Verify activity
4. **Don't ignore project conventions** - Respect CLAUDE.md guidelines
5. **Don't break functionality** - Ensure fixes are safe
6. **Don't over-engineer** - Consider cost/benefit ratio
7. **Don't skip type checks** - `mypy`/type hints are documentation and catch real defects
8. **Don't ignore dependency weight** - transitive deps add import/startup cost to every run

## Performance Optimization

For large codebases:
- **Parallel processing**: Analyze multiple files simultaneously
- **Batch operations**: Group similar checks together
- **Selective scanning**: Focus on changed files first
- **Cache documentation**: Reuse Context7 lookups
- **Progressive reporting**: Provide interim results
