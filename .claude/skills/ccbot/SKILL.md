```markdown
# ccbot Development Patterns

> Auto-generated skill from repository analysis

## Overview
This skill teaches the core development patterns and conventions used in the `ccbot` Python codebase. You'll learn about file naming, import/export styles, commit conventions, and how to structure and run tests. This guide is ideal for contributors looking to maintain consistency and quality in the project.

## Coding Conventions

### File Naming
- Use **snake_case** for all Python files.
  - **Example:**  
    ```python
    # Good
    my_module.py

    # Bad
    MyModule.py
    myModule.py
    ```

### Import Style
- Use **relative imports** within the package.
  - **Example:**  
    ```python
    # Good
    from .utils import helper_function

    # Bad
    from utils import helper_function
    ```

### Export Style
- Use **named exports** (i.e., define `__all__` in modules to specify public API).
  - **Example:**  
    ```python
    __all__ = ['main_function', 'HelperClass']
    ```

### Commit Messages
- Follow **conventional commit** style.
- Use the `fix` prefix for bug fixes.
- Keep commit messages concise (average 38 characters).
  - **Example:**  
    ```
    fix: handle edge case in message parser
    ```

## Workflows

### Code Contribution
**Trigger:** When adding new features or fixing bugs  
**Command:** `/contribute`

1. Create a new branch for your change.
2. Follow snake_case naming for files.
3. Use relative imports in your modules.
4. Add or update `__all__` for exports.
5. Write tests (see Testing Patterns).
6. Commit using the `fix` prefix if it's a bug fix.
7. Submit a pull request.

### Commit Formatting
**Trigger:** When making a commit  
**Command:** `/commit-format`

1. Write commit messages using the conventional commit style.
2. Use `fix:` for bug fixes.
3. Limit the message to around 38 characters.

   **Example:**
   ```
   fix: correct typo in response handler
   ```

## Testing Patterns

- **Framework:** Unknown (no explicit framework detected)
- **File Pattern:** Test files use the `*.test.ts` pattern (TypeScript), suggesting some cross-language or external test suite.
- **Best Practice:** Place tests in files ending with `.test.ts`. Ensure all new features and bug fixes are covered by corresponding tests.

  **Example:**
  ```
  my_feature.test.ts
  ```

## Commands
| Command         | Purpose                                    |
|-----------------|--------------------------------------------|
| /contribute     | Steps for contributing code                |
| /commit-format  | How to format commit messages              |
```
