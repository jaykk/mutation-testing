# Mutation Testing in Python

## What is Mutation Testing?

Mutation testing is a powerful technique for evaluating the quality of your test suite. Unlike traditional code coverage metrics that only measure which lines of code are executed, mutation testing assesses whether your tests can actually detect bugs.

The process works by:
1. Creating small modifications (mutations) to your source code
2. Running your test suite against each mutation
3. Checking if the tests fail (kill the mutant) or pass (mutant survives)

If a mutant survives, it indicates a gap in your test coverage - your tests didn't catch the code change, suggesting they might not catch real bugs either.

## Why Use Mutation Testing?

**Traditional code coverage can be misleading:**
```python
def add(a, b):
    return a + b

def test_add():
    result = add(2, 3)  # This achieves 100% line coverage
    # But no assertion! Test passes regardless of correctness
```

**Mutation testing reveals the problem:**
- Mutant: `return a - b` → Test still passes (mutant survives!)
- This reveals that the test isn't actually validating behavior

**Benefits:**
- ✅ Identifies weak or missing test assertions
- ✅ Finds gaps in test logic and edge cases
- ✅ Improves overall test suite quality
- ✅ Provides confidence in your tests' ability to catch real bugs

## How It Works: AST Parsing

This setup uses **Abstract Syntax Tree (AST) parsing** to identify and mutate code. Instead of using simple text-based find-and-replace, AST parsing:

- **Understands code structure** - Parses Python code into a tree representation
- **Precisely targets code elements** - Identifies specific operators, functions, and statements
- **Maintains valid syntax** - Ensures all mutations produce syntactically correct Python code
- **Enables smart matching** - Can locate code pieces based on their semantic meaning, not just text patterns

For example, when mutating `a + b`, the AST parser recognizes this as a `BinOp` (binary operation) node with the `Add` operator, allowing it to intelligently replace it with `Sub`, `Mult`, etc., while understanding the context.

*More technical details on the AST parsing implementation will be provided in future updates.*

## Popular Python Mutation Testing Tools

### 1. mutmut
Fast and easy-to-use mutation testing tool with minimal configuration.

**Installation:**
```bash
pip install mutmut
```

**Basic Usage:**
```bash
# Run mutation testing
mutmut run

# Show results
mutmut results

# Show surviving mutants
mutmut show
```

### 2. Cosmic Ray
Comprehensive mutation testing tool with support for multiple execution engines.

**Installation:**
```bash
pip install cosmic-ray
```

**Basic Usage:**
```bash
# Initialize
cosmic-ray init config.toml

# Run mutation testing
cosmic-ray exec config.toml session.sqlite

# View report
cr-report session.sqlite
```

### 3. MutPy
Supports Python 3.x with detailed HTML reports.

**Installation:**
```bash
pip install mutpy
```

**Basic Usage:**
```bash
mut.py --target mymodule --unit-test tests --report-html report
```

## Getting Started with mutmut

### Step 1: Install mutmut
```bash
pip install mutmut
```

### Step 2: Run mutation testing
```bash
# Run on entire codebase
mutmut run

# Run on specific paths
mutmut run --paths-to-mutate=src/

# Run with specific test command
mutmut run --tests-dir=tests/ --runner="pytest -x"
```

### Step 3: Analyze results
```bash
# View summary
mutmut results

# Show specific mutant
mutmut show <mutant-id>

# Generate HTML report
mutmut html
```

## Example: Complete Workflow

### Original Code (calculator.py)
```python
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b

def is_positive(num):
    return num > 0
```

### Weak Tests (test_calculator.py)
```python
def test_add():
    result = add(2, 3)
    assert result  # Weak assertion!

def test_subtract():
    subtract(5, 3)  # No assertion at all!

def test_multiply():
    assert multiply(2, 3) == 6  # Good assertion

def test_is_positive():
    assert is_positive(1) == True
    # Missing: test for negative numbers, zero
```

### Running Mutation Testing
```bash
$ mutmut run
- Mutation testing starting -

These are the mutations:
-----------------------------
SURVIVED (weak test): calculator.py:2 - Changed + to -
SURVIVED (no test): calculator.py:5 - Changed - to +
KILLED (good test): calculator.py:8 - Changed * to /
SURVIVED (missing edge case): calculator.py:11 - Changed > to >=

Mutation testing complete!
Survived: 3
Killed: 1
Mutation Score: 25%
```

### Improved Tests
```python
def test_add():
    assert add(2, 3) == 5  # Specific assertion
    assert add(-1, 1) == 0
    assert add(0, 0) == 0

def test_subtract():
    assert subtract(5, 3) == 2  # Added assertion
    assert subtract(0, 5) == -5

def test_multiply():
    assert multiply(2, 3) == 6
    assert multiply(-2, 3) == -6
    assert multiply(0, 5) == 0

def test_is_positive():
    assert is_positive(1) == True
    assert is_positive(-1) == False  # Added negative case
    assert is_positive(0) == False   # Added zero case
```

### After Improvements
```bash
$ mutmut run
Mutation testing complete!
Survived: 0
Killed: 10
Mutation Score: 100%
```

## Common Mutations

Mutation testing tools typically apply these types of mutations:

| Category | Original | Mutated |
|----------|----------|---------|
| **Arithmetic** | `+` | `-`, `*`, `/` |
| | `-` | `+`, `*`, `/` |
| | `*` | `+`, `-`, `/`, `**` |
| **Comparison** | `>` | `>=`, `<`, `==` |
| | `==` | `!=`, `<`, `>` |
| | `<=` | `<`, `>` |
| **Boolean** | `and` | `or` |
| | `or` | `and` |
| | `True` | `False` |
| **Unary** | `+x` | `-x` |
| | `not x` | `x` |
| **Return** | `return x` | `return None` |
| **Constants** | `0` | `1` |
| | `"text"` | `""` |

## Best Practices

### 1. Start Small
Begin with critical modules rather than your entire codebase:
```bash
mutmut run --paths-to-mutate=src/core/
```

### 2. Set Realistic Goals
- **80-90% mutation score**: Excellent
- **60-80%**: Good
- **<60%**: Needs improvement

Don't aim for 100% - some mutants are equivalent or impractical to kill.

### 3. Handle Equivalent Mutants
Some mutants are functionally equivalent to the original:
```python
# Original
if x >= 0:
    return True
return False

# Mutant (equivalent if x is always integer)
if x > -1:
    return True
return False
```

Mark these as equivalent:
```bash
mutmut show <id>  # Review the mutant
# If equivalent, exclude it
```

### 4. Integrate with CI/CD
```yaml
# .github/workflows/mutation-test.yml
name: Mutation Testing

on: [pull_request]

jobs:
  mutation-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install mutmut pytest
          pip install -r requirements.txt
      - name: Run mutation tests
        run: |
          mutmut run
          mutmut results
```

### 5. Focus on High-Value Code
Prioritize mutation testing for:
- Business logic
- Security-critical code
- Complex algorithms
- Bug-prone areas

Skip mutation testing for:
- UI/presentation layers
- Simple getters/setters
- Framework boilerplate

## Configuration Example

### mutmut configuration (setup.cfg)
```ini
[mutmut]
paths_to_mutate=src/
backup=False
runner=pytest -x --tb=short
tests_dir=tests/
dict_synonyms=Struct, NamedStruct
```

### Cosmic Ray configuration (cosmic-ray.toml)
```toml
[cosmic-ray]
module-path = "mypackage"
timeout = 10.0
excluded-modules = ["tests"]
test-command = "pytest tests"

[cosmic-ray.distributor]
name = "local"
```

## Interpreting Results

### Mutation Score Formula
```
Mutation Score = (Killed Mutants / Total Mutants) × 100%
```

### Result Categories

1. **KILLED** ✅ - Test caught the mutation (good!)
2. **SURVIVED** ⚠️ - Mutation not detected (test gap!)
3. **TIMEOUT** ⏱️ - Test took too long (possible infinite loop)
4. **SUSPICIOUS** 🤔 - Unexpected test behavior
5. **SKIPPED** ⏭️ - Mutation not applied

### Action Items by Result

| Result | Action |
|--------|--------|
| High survival rate | Add assertions to tests |
| Many timeouts | Fix mutations causing infinite loops |
| Low coverage | Write more test cases |
| Specific survivors | Add edge case tests |

## Advanced Topics

### Custom Mutations
Some tools allow defining custom mutation operators for domain-specific testing.

### Mutation Testing Metrics
- **Mutation Score**: Overall percentage of killed mutants
- **Mutation Coverage**: Percentage of mutants covered by tests
- **Test Effectiveness**: Ability to detect specific mutation types

### Performance Optimization
- Use parallel execution: `mutmut run --use-parallel`
- Cache results between runs
- Test only changed code in CI
- Use sampling for large codebases

## Resources

- [mutmut Documentation](https://mutmut.readthedocs.io/)
- [Cosmic Ray Documentation](https://cosmic-ray.readthedocs.io/)
- [Mutation Testing: A Comprehensive Survey](https://ieeexplore.ieee.org/document/8633556)
- [PIT Mutation Testing](https://pitest.org/) - Inspiration from Java ecosystem

## Comparison: Code Coverage vs Mutation Testing

| Aspect | Code Coverage | Mutation Testing |
|--------|---------------|------------------|
| **Measures** | Lines executed | Bug detection ability |
| **Speed** | Fast | Slower |
| **Cost** | Low | Higher |
| **Insights** | Quantity of testing | Quality of testing |
| **False confidence** | High risk | Low risk |
| **Best use** | Quick feedback | Deep validation |

## Conclusion

Mutation testing is a valuable tool for improving test suite quality. While it requires more computational resources than traditional coverage, the insights it provides about test effectiveness are invaluable.

**Remember:**
- Code coverage tells you what code is tested
- Mutation testing tells you if your tests are actually effective

Start small, focus on critical code, and use mutation testing as part of your quality assurance strategy!

## Contributing

Have tips or examples to share? Contributions are welcome!

## License

This guide is provided as-is for educational purposes.
