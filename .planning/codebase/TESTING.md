# Testing Patterns

**Analysis Date:** 2026-02-04

## Test Framework

**Status:** Not detected

No dedicated test framework (pytest, unittest, nose) is configured in the repository. The codebase contains no test files (no `test_*.py` or `*_test.py` files found).

**Testing Approach:** Validation-based design

Rather than unit tests, the codebase employs input validation and structured result objects that allow consumers to determine success/failure:

- `ValidationResult` dataclass (`validators.py` lines 15-41) returns structured validation output with errors, warnings, and suggestions
- `CheckResult` dataclass (`check_environment.py` lines 22-29) returns individual check results with pass/fail status and remediation steps
- `EnvironmentReport` dataclass (`check_environment.py` lines 32-44) aggregates multiple checks

## Testing Philosophy

The codebase prioritizes **fail-safe validation over automated tests**. Key characteristics:

1. **Pre-execution Validation:** Errors are caught before operations start
   - `generate_samplesheet.py` lines 74-94: Validates configuration, input types, and file presence before processing
   - `validators.py` lines 57-164: Complete samplesheet validation with detailed error/warning accumulation

2. **Structured Error Reporting:** Actionable error messages with recovery steps
   - `check_environment.py` CheckResult includes optional `fix` field with remediation instructions
   - Example: Docker permission error includes `fix="sudo usermod -aG docker $USER && newgrp docker"`

3. **Incremental Error Collection:** Errors don't stop validation
   - `validators.py` lines 73-163: Processes all rows and collects all errors, not stopping at first failure
   - Returns comprehensive `ValidationResult` with complete error list

4. **Pipeline-Specific Validation:** Modular validation logic
   - `validators.py` lines 154-157: Dispatches to `_validate_sarek_specific()` or `_validate_atacseq_specific()`
   - Allows different validation rules per pipeline configuration

## Testing Patterns in Use

### Input Validation Pattern

**Location:** `validators.py`, `check_environment.py`, `generate_samplesheet.py`

**Pattern:**
```python
def validate_something(input_data) -> ValidationResult:
    errors = []
    warnings = []
    suggestions = []

    # Check 1
    if condition:
        errors.append("descriptive error")

    # Check 2
    if warning_condition:
        warnings.append("informational warning")

    return ValidationResult(
        valid=len(errors) == 0,
        errors=errors,
        warnings=warnings,
        suggestions=suggestions
    )
```

**Examples:**
- `validators.py` lines 57-164: `validate_samplesheet()` - validates rows against pipeline config
- `check_environment.py` lines 47-110: `check_docker()` - validates Docker installation and permissions
- `generate_samplesheet.py` lines 52-131: `generate_samplesheet()` - validates input and generates samplesheet

### Result Object Pattern

**Dataclass-based structured returns:**

`ValidationResult` in `validators.py` lines 15-41:
```python
@dataclass
class ValidationResult:
    """Result of samplesheet validation."""
    valid: bool
    errors: List[str] = field(default_factory=list)
    warnings: List[str] = field(default_factory=list)
    suggestions: List[str] = field(default_factory=list)

    def __bool__(self):
        return self.valid

    def summary(self) -> str:
        """Generate human-readable summary."""
        lines = []
        if self.errors:
            lines.append("Errors:")
            for e in self.errors:
                lines.append(f"  - {e}")
        # ... warnings and suggestions ...
        return "\n".join(lines)
```

**Usage Pattern:**
```python
validation = validate_samplesheet(rows, pipeline)
if validation:  # __bool__ checks valid flag
    # Process samplesheet
else:
    print(validation.summary())  # Print all issues
```

### Conditional Validation Pattern

**Location:** `validators.py` lines 101-104

**Pattern:** Skip validation based on conditions
```python
if col_config and "condition" in col_config:
    # Simple condition check - skip for now
    # Full implementation would evaluate conditions
    pass
```

Notes:
- Conditions are defined in config but not fully implemented
- Allows future expansion of conditional validation rules

### Enum/Allowed Value Validation

**Location:** `validators.py` lines 122-131

**Pattern:**
```python
if col_name in row and row[col_name] and "allowed" in col_config:
    value = row[col_name]
    allowed = col_config["allowed"]
    if value not in allowed:
        errors.append(
            f"Row {row_num}: Invalid value '{value}' for '{col_name}'. "
            f"Allowed: {allowed}"
        )
```

**Data Source:** Configuration file driven
- Config loaded from YAML: `validators.py` lines 44-54
- Allowed values specified in pipeline config under `samplesheet.columns`

### File Existence Validation

**Location:** `validators.py` lines 113-119

**Pattern:**
```python
for col_name in ["fastq_1", "fastq_2", "bam", "bai"]:
    if col_name in row and row[col_name]:
        path = row[col_name]
        if not os.path.exists(path):
            errors.append(f"Row {row_num}: File not found: {path}")
        elif not os.path.isfile(path):
            errors.append(f"Row {row_num}: Not a file: {path}")
```

### Environment Check Pattern

**Location:** `check_environment.py` lines 47-240

**Pattern:** Subprocess execution with signal handling and timeout
```python
def check_docker() -> CheckResult:
    if not shutil.which("docker"):
        return CheckResult(
            name="Docker",
            passed=False,
            message="Docker not found in PATH",
            fix="Install Docker: https://docs.docker.com/get-docker/"
        )

    try:
        result = subprocess.run(
            ["docker", "info"],
            capture_output=True,
            text=True,
            timeout=15
        )

        if result.returncode != 0:
            # Parse stderr for specific errors
            stderr_lower = result.stderr.lower()
            if "permission denied" in stderr_lower:
                return CheckResult(
                    name="Docker",
                    passed=False,
                    message="Docker permission denied",
                    fix="sudo usermod -aG docker $USER && newgrp docker"
                )
            # ... more specific checks ...

        return CheckResult(name="Docker", passed=True, message="...")

    except subprocess.TimeoutExpired:
        return CheckResult(...)
    except Exception as e:
        return CheckResult(...)
```

### Network Connectivity Testing

**Location:** `ncbi_utils.py` lines 60-97

**Pattern:** Graceful degradation with retry logic
```python
def check_network_access() -> Tuple[bool, str]:
    test_urls = [
        ("NCBI Entrez", "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/einfo.fcgi"),
        ("NCBI FTP", "https://ftp.ncbi.nlm.nih.gov/"),
        ("ENA API", "https://www.ebi.ac.uk/ena/portal/api/"),
    ]

    results = []
    for name, url in test_urls:
        try:
            if HAS_REQUESTS:
                response = requests.get(url, timeout=10)
                success = response.status_code < 400
            else:
                # Fallback to urllib if requests not available
                req = Request(url, headers={'User-Agent': 'geo-sra-skill/1.0'})
                with urlopen(req, timeout=10) as response:
                    success = True
            results.append((name, success, None))
        except Exception as e:
            results.append((name, False, str(e)))

    # Return summary
    all_success = all(r[1] for r in results)
    # ... format message ...
    return all_success, message
```

## Error Detection & Reporting

### Multi-Level Validation Results

Errors, warnings, and suggestions are tracked separately:

**Errors (fatal):** `validation.valid = False` if any errors present
- Invalid values, missing required fields, file not found

**Warnings (non-fatal):** Informational issues that don't prevent processing
- Single-end data detected (requires R2 for pipeline)
- Duplicate sample names
- Tumor without matched normal

**Suggestions:** Actionable recommendations
- "Duplicates may be intentional... Verify sample grouping is correct."
- "For patient X: Add a normal sample or use tumor-only mode."

Example from `validators.py` lines 141-151:
```python
# Check for duplicate samples
samples = [r.get(sample_col, "") for r in rows]
duplicates = [s for s in set(samples) if samples.count(s) > 1]
if duplicates:
    warnings.append(f"Duplicate sample names: {duplicates}")
    suggestions.append(
        "Duplicates may be intentional (multi-lane sequencing). "
        "Verify sample grouping is correct."
    )
```

### Batch Validation

All rows validated before returning results (no early exit):
- `validators.py` lines 93-164: Loop through all rows, accumulating errors
- Allows reporting ALL issues at once vs one issue per run
- Efficient for large samplesheets

Example flow from `generate_samplesheet.py` lines 91-129:
```python
# Discover and validate all files
files = discover_files(input_dir, input_type)  # May process hundreds of files

# Generate rows from discovered files
rows = []  # Build complete list

# Validate entire samplesheet at once
validation = validate_samplesheet(rows, pipeline)

if not validation:
    print("\nValidation errors:")
    for error in validation.errors:
        print(f"  - {error}")
    # ... handle errors ...
```

## Coverage & Tested Components

**Validated/Tested (through input validation):**
- File discovery and pattern matching: `file_discovery.py` tested via validation
- Sample metadata inference: `sample_inference.py` patterns used in validation
- Samplesheet structure: `validators.py` comprehensive checks
- Environment prerequisites: `check_environment.py` thorough checks
- Metadata parsing: `validate_asm.py` exhaustive validation

**Not Formally Tested:**
- Mathematical algorithms in QC metrics (rely on library correctness)
- scvi-tools model training (rely on library behavior)
- File I/O operations (rely on subprocess/API correctness)

## Testing Recommendations

The validation-first approach works well but could be enhanced:

1. **Add unit tests for parsing logic:**
   - `sample_inference.py` pattern matching with test cases
   - `file_discovery.py` filename parsing edge cases
   - Example patterns to test: Illumina standard vs. non-standard naming

2. **Test error message accuracy:**
   - Ensure error messages point to actual problems
   - Validate suggestion text is actionable

3. **Integration tests for real files:**
   - Test against actual FASTQ/BAM files
   - Validate against real pipeline configurations

4. **Mock network tests:**
   - Mock NCBI/ENA responses to test fetch functions
   - Test timeout and retry behavior

---

*Testing analysis: 2026-02-04*
