---
name: Code Reviewer
description: This skill shows how to perform code review on a specific code set
instructions: |
  When performing a code review, follow this systematic approach:

  ## 1. Understand the Context
  - Read the commit message or PR description to understand the intent
  - Check what files were changed: `git diff --name-only`
  - Review the full diff: `git diff` or `git diff [base-branch]...HEAD`

  ## 2. Correctness Review
  - **Logic errors**: Does the code do what it's supposed to? Check edge cases, boundary conditions, off-by-one errors
  - **Type safety**: Are types used correctly? Any unsafe casts or missing null checks?
  - **Error handling**: Are errors caught and handled appropriately? Could exceptions crash the app?
  - **Race conditions**: For concurrent code, check for data races, deadlocks, or improper synchronization
  - **Security vulnerabilities**: Check for XSS, SQL injection, command injection, insecure data storage, exposed secrets

  ## 3. Code Quality Review
  - **Simplification**: Can complex logic be simplified? Are there unnecessary abstractions or premature optimizations?
  - **Reusability**: Is there duplicated code that could be extracted? Are there existing utilities being reimplemented?
  - **Naming**: Are variables, functions, and types clearly named? Do names accurately reflect their purpose?
  - **Comments**: Are there comments explaining non-obvious WHY (not WHAT)? Are there misleading or outdated comments?
  - **Performance**: Are there obvious inefficiencies? Unnecessary loops, N+1 queries, missing indexes?

  ## 4. Testing Review
  - Are there tests for the new functionality?
  - Do tests cover edge cases and error paths?
  - Are tests clear and maintainable?

  ## 5. Standards and Consistency
  - Does the code follow project conventions and style?
  - Is it consistent with surrounding code patterns?
  - Are there linting or formatting issues?

  ## 6. Provide Feedback
  Structure your review as:
  - **Critical issues**: Bugs, security vulnerabilities, data loss risks (must fix)
  - **Important issues**: Logic errors, poor error handling, significant performance problems (should fix)
  - **Suggestions**: Simplifications, reusability improvements, naming improvements (nice to have)
  
  For each issue:
  - Reference the file and line number: `file.js:42`
  - Explain the problem clearly
  - Suggest a specific fix when possible
  - Explain why it matters

  ## 7. Balance
  - Focus on issues that actually matter - avoid nitpicking formatting if linters handle it
  - Praise good decisions and clean code when you see it
  - Consider the scope: a quick fix doesn't need architectural review
  - Don't ask for changes that could be separate follow-up work 