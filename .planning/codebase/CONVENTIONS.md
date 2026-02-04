# Coding Conventions

**Analysis Date:** 2026-02-04

## Naming Patterns

**Files:**
- Snake case for all Python files: `generate_samplesheet.py`, `model_utils.py`, `validate_asm.py`
- Descriptive names that indicate purpose: utilities grouped in `utils/` subdirectory
- Script files start with shebang: `#!/usr/bin/env python3`
- Utility modules grouped by domain: `file_discovery.py`, `validators.py`, `sample_inference.py` all in `utils/`

**Functions:**
- Snake case for all function names: `calculate_qc_metrics()`, `extract_sample_info()`, `validate_samplesheet()`
- Helper/private functions prefixed with underscore: `_rate_limit_ncbi()`, `_validate_sarek_specific()`, `_get_pattern_score()`
- Function names describe their action: verb + noun pattern: `detect_outliers_mad()`, `filter_cells()`, `infer_tumor_normal_status()`
- Single letter shorthand avoided except in mathematical contexts: `adata` (standard AnnData convention), `n_mads` (standard bioinformatics)

**Variables:**
- Snake case throughout: `sample_name`, `input_dir`, `output_file`, `validation_result`
- Plural names for collections: `files`, `rows`, `errors`, `warnings`, `suggestions`
- Standard domain abbreviations: `adata` for AnnData objects, `df` for dataframes
- Boolean prefixes: `is_valid`, `has_requests`, `verbose`

**Types & Classes:**
- Pascal case for all classes: `ValidationResult`, `CheckResult`, `EnvironmentReport`, `FileInfo`
- Dataclass usage preferred for result containers: See `validators.py` line 15-24, `check_environment.py` line 22-36

**Constants:**
- All caps with underscores: `R1_PATTERNS`, `R2_PATTERNS`, `TUMOR_KEYWORDS`, `NORMAL_KEYWORDS`, `LANE_PATTERN`, `PATIENT_PATTERNS`, `REPLICATE_PATTERNS`
- Configuration/magic values defined at module level before functions: `sample_inference.py` lines 13-75
- Version strings in constants: `ASM_SPEC_VERSION = "2024-12"` (validate_asm.py line 33)

## Code Style

**Formatting:**
- No explicit formatter configured (black/autopep8 not enforced)
- Standard Python line length appears to be around 100 characters (most files respect this)
- 4-space indentation consistently applied
- Module-level docstrings always present: Triple-quoted strings at top of file describing purpose and usage
- Blank line between imports and code, between class definition and methods

**Import Organization:**
Order observed in codebase:
1. Standard library imports: `import os`, `import sys`, `import json`, `import logging`
2. Third-party imports: `import numpy as np`, `import scanpy as sc`, `import anndata as ad`, `import yaml`
3. Typing imports: `from typing import Dict, List, Optional, Tuple`
4. Local imports: `from utils.validators import ...`

Example from `generate_samplesheet.py` lines 17-36:
```python
import argparse
import os
import sys
from pathlib import Path
from typing import Dict, List, Optional, Tuple

import yaml

# Add parent directory to path for utils import
sys.path.insert(0, str(Path(__file__).parent))

from utils.file_discovery import discover_files, detect_input_type, find_index_file
from utils.sample_inference import (...)
from utils.validators import validate_samplesheet, ValidationResult
```

**Path Aliases:**
- No aliases configured; absolute imports from script directory
- Relative path manipulation using `pathlib.Path` preferred: `Path(__file__).parent`, `Path(__file__).parent.parent`

## Docstring Format

**Style:** NumPy docstring format throughout

**Pattern:**
```python
def function_name(param1: str, param2: int) -> Dict:
    """
    Short description (one line).

    Longer description if needed. More context about what
    the function does and edge cases.

    Parameters
    ----------
    param1 : str
        Description of param1
    param2 : int
        Description of param2

    Returns
    -------
    Dict
        Description of return value

    Examples
    --------
    >>> result = function_name("value", 42)
    """
```

Examples:
- `qc_core.py` lines 18-38 (calculate_qc_metrics)
- `validators.py` lines 62-72 (validate_samplesheet)
- `model_utils.py` lines 44-69 (prepare_adata)
- `cluster_embed.py` lines 18-44 (cluster_and_embed)

## Error Handling

**Pattern:** Explicit exception handling with informative messages

**Approach:**
- Specific exception types caught: `ValueError`, `FileNotFoundError`, `PermissionError`, `TimeoutExpired`
- Generic fallback: `except Exception as e:` used only when appropriate
- Error messages include context: `f"Row {row_num}: Missing required column '{col_name}'"`
- Structured error tracking via result objects: See `check_environment.py` CheckResult dataclass

**Examples:**
- `generate_samplesheet.py` lines 78-94: ValueError for unknown pipeline, try/except for file operations
- `validators.py` lines 96-110: Conditional checks with descriptive error messages accumulated in list
- `check_environment.py` lines 47-110: Try/except with timeout handling and recovery suggestions

**Patterns Not Seen:**
- No global exception handlers (try/except at module level)
- No silent failures - errors always reported or re-raised
- Bare `except:` avoided except in fallback scenarios (check_environment.py lines 214-225)

## Logging

**Framework:** Python `logging` module

**Setup Pattern:**
```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)
```

Seen in `ncbi_utils.py` lines 23-29

**Usage:**
- `logger.debug()` for diagnostic info
- `logger.info()` for significant events
- `logger.warning()` for potentially harmful situations
- Print statements used for user-facing output (validation results, progress)
- No mixing of print and logging to stderr

**Example from ncbi_utils.py:**
```python
logger.debug("requests not installed - using urllib fallback")
```

## Comments

**When to Comment:**
- Algorithm explanation for non-obvious logic: `sample_inference.py` lines 13-30 (pattern explanations with priority scores)
- Business logic reasons: `generate_samplesheet.py` lines 67-76 (parameter documentation for pipeline config)
- Workarounds and their rationale: `validate_asm.py` line 22-29 (warnings vs errors for unknown techniques)
- Links to external standards: `qc_core.py` line 6, `validate_asm.py` line 25

**When NOT to Comment:**
- Self-evident code: `if col_name not in row:` needs no explanation
- Docstrings replace inline comments for function behavior

**Comment Style:**
- Inline comments after code: `# Account for header row` (validators.py line 94)
- Block comments above logical sections (rare - docstrings preferred)
- No commented-out code committed

## Function Design

**Size:**
- Most functions 20-50 lines
- Larger functions (100+ lines): `validate_asm()` in validate_asm.py, `check_docker()` in check_environment.py
- Longer functions handle distinct, tightly-coupled logic

**Parameters:**
- Minimal positional parameters (usually 1-3)
- Additional parameters use defaults: `def prepare_adata(adata, batch_key: Optional[str] = None, n_top_genes: int = 2000, ...)`
- Type hints always present in function signatures (seen consistently across all files)
- Return type hints always specified: `-> Dict`, `-> ValidationResult`, `-> bool`

**Return Values:**
- Single return value preferred
- Tuples for multiple related values: `(output_path, validation_result)` from `generate_samplesheet()`
- None as sentinel for missing/failed cases: `Optional[Dict]` return types
- Result objects for complex return data: `ValidationResult` dataclass with `errors`, `warnings`, `suggestions` lists
- Consistent return on all paths (no implicit None returns)

## Module Design

**Exports:**
- Barrel file pattern in `__init__.py`: `utils/__init__.py` lines 1-10 re-exports public functions
- Private functions prefixed with underscore not re-exported
- Direct function/class imports: `from .validators import validate_samplesheet, ValidationResult`

**Barrel Files:**
Usage in `nextflow-development/scripts/utils/__init__.py`:
```python
from .ncbi_utils import (fetch_geo_metadata, fetch_sra_metadata, ...)
from .file_discovery import discover_files, FileInfo, count_files_by_type
from .sample_inference import (extract_sample_info, infer_tumor_normal_status, ...)
from .validators import validate_samplesheet, ValidationResult
```

Allows consumers to import: `from utils import validate_samplesheet` instead of `from utils.validators import ...`

## Type Hints

**Usage:** Universal throughout codebase

**Pattern:**
```python
def function(param1: str, param2: List[Dict], opt: Optional[int] = None) -> Tuple[str, ValidationResult]:
```

**Standard typing imports:**
```python
from typing import Dict, List, Optional, Tuple, Any
```

**Common patterns:**
- Optional for nullable: `Optional[str]`, `Optional[Dict]`
- Collections: `List[Dict]`, `List[str]`, `Dict[str, int]`
- Tuples: `Tuple[str, int]`, `Tuple[bool, str]`
- Union when multiple types: Not heavily used - Optional preferred for None cases

## Data Structures

**Dataclasses:** Preferred for structured return types and configuration containers

Examples:
- `ValidationResult` in `validators.py`: Holds valid (bool), errors (List), warnings (List), suggestions (List)
- `CheckResult` in `check_environment.py`: Holds name, passed, message, details, fix
- `EnvironmentReport` in `check_environment.py`: Holds ready (bool) and list of CheckResults
- `FileInfo` in `file_discovery.py`: Holds file metadata

Pattern from validators.py lines 15-24:
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
        # ...
```

**Dictionaries:** Used for flexible data (file rows, metadata, config)
- Often typed: `Dict[str, str]`, `Dict[str, Any]`
- Built from parsed YAML/CSV for configuration

## Configuration & Constants

**Constants defined at module level** before functions:
- Pattern lists with priority tuples: `R1_PATTERNS = [(pattern, score), ...]`
- Keyword lists for inference: `TUMOR_KEYWORDS = [r'pattern1', r'pattern2', ...]`
- Feature flags: `ASM_SPEC_VERSION = "2024-12"`

Example from sample_inference.py lines 13-59:
```python
# R1/R2 patterns with priority scores (higher = more confident)
R1_PATTERNS = [
    (r'_R1_\d{3}', 10),      # _R1_001 (Illumina standard)
    (r'_R1[_.]', 8),         # _R1. or _R1_
    # ...
]
```

**Configuration loading:** Via YAML files with path resolution
- `generate_samplesheet.py` lines 39-49: Relative path from script location
- Environment-specific: Not seen; hardcoded paths to config/ directory

---

*Convention analysis: 2026-02-04*
