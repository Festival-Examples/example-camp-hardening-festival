# Festival Rules: camp-hardening

## Purpose

This document defines the principles, standards, and practices that all workers
must follow throughout this festival. Reference these rules before starting any
task and during task execution to ensure consistent quality.

## Engineering Excellence

### Code Quality

- **Prefer Refactoring Over Rewriting**: Only rewrite code when it's
  fundamentally flawed or unmaintainable
- **Follow Existing Patterns**: Study and follow established patterns in the
  codebase before introducing new ones
- **Apply SOLID Principles**:
  - Single Responsibility: Each class/function should have one reason to change
  - Open/Closed: Open for extension, closed for modification
  - Liskov Substitution: Derived classes must be substitutable for base classes
  - Interface Segregation: Many specific interfaces are better than one general
    interface
  - Dependency Inversion: Depend on abstractions, not concretions

### Design Principles

- **KISS (Keep It Simple)**: Choose simple solutions over complex ones
- **YAGNI (You Aren't Gonna Need It)**: Don't add functionality until it's
  needed
- **DRY (Don't Repeat Yourself)**: Eliminate duplication through abstraction
- **Composition Over Inheritance**: Prefer composition for code reuse

### Code Organization

- Functions should be under 50 lines
- Files should be under 500 lines
- Classes should have a single, well-defined purpose
- Use dependency injection instead of global state
- Separate concerns into distinct modules

## Quality Standards

### Testing Requirements

- [ ] Write tests for all new functionality
- [ ] Maintain or improve existing code coverage
- [ ] Include unit tests for business logic
- [ ] Add integration tests for system boundaries
- [ ] Filesystem-mutating UNIT tests follow camp's existing host `t.TempDir` convention; integration coverage of real git/filesystem effects goes in the container harness under `tests/integration/` (see Festival-Specific Rules)
- [ ] Test error cases before happy paths
- [ ] Include tests for edge cases

### Code Review Checklist

- [ ] Code follows language conventions and style guide
- [ ] No commented-out code or debug statements
- [ ] Clear, self-documenting variable and function names
- [ ] Appropriate error handling with context
- [ ] No security vulnerabilities introduced
- [ ] Performance implications considered

### Documentation Standards

- [ ] Update documentation alongside code changes
- [ ] Include clear commit messages explaining "why" not just "what"
- [ ] Document architectural decisions and trade-offs
- [ ] Add inline comments only for complex algorithms
- [ ] Update API documentation for interface changes
- [ ] Include examples in documentation where helpful

## Development Process

### Task and Sequence Numbering

- **Sequential Execution**: Both sequences and tasks must be prepended with
  numbers when they must be completed in order (e.g., `1_task_name.md`,
  `2_task_name.md`)
- **Parallel Tasks**: Tasks within a sequence that can be worked on in parallel
  may use the same number (e.g., `1_task_a.md`, `1_task_b.md`, `1_task_c.md`)
- **Verification Workflow**: Every sequence must include:
  - `N_testing_and_verify.md` - Testing and verification task (where N is the
    next number after implementation tasks)
  - `N+1_code_review.md` - Code review task
  - `N+2_review_results_update_tasks_iterate_if_needed.md` - Final review and
    iteration task
- **Results Directory**: Testing results and code review documents must be
  placed in a `results/` subdirectory within the sequence directory

### Before Starting a Task

1. Read and understand the task requirements completely
2. Review relevant existing code and documentation
3. Identify dependencies and potential impacts
4. Plan your approach before coding
5. Check if similar functionality exists to reuse
6. Note the task number to understand execution order and dependencies

### During Development

- Create small, focused commits with clear messages
- Run tests frequently during development
- Use version control effectively (branch, commit, push)
- Keep pull requests focused on a single logical change
- Refactor as you go to maintain code quality

### Before Marking Complete

- [ ] All tests pass locally
- [ ] Linters and formatters have been run
- [ ] Type checking passes (if applicable)
- [ ] Documentation is updated
- [ ] Code has been self-reviewed
- [ ] Security implications considered
- [ ] Performance impact assessed
- [ ] Backward compatibility maintained (unless approved to break)

## Decision Making

### When to Escalate

- Breaking changes to public APIs
- Significant architectural decisions
- Security-related changes
- Performance trade-offs
- Introduction of new dependencies
- Deviation from established patterns

### Technical Debt

- Document technical debt when created
- Include TODO comments with issue numbers
- Prefer incremental improvements over big rewrites
- Balance perfection with delivery

## Security Principles

- Never commit secrets, keys, or credentials
- Validate all inputs
- Sanitize all outputs
- Follow principle of least privilege
- Consider security implications in all changes
- Keep dependencies up to date

## Performance Guidelines

- Measure before optimizing
- Profile to find actual bottlenecks
- Consider algorithmic improvements before micro-optimizations
- Cache expensive operations appropriately
- Be mindful of memory usage
- Consider scalability implications

## Collaboration

- Communicate blockers early
- Ask for help when stuck for more than 30 minutes
- Share knowledge through documentation
- Be open to feedback and suggestions
- Help team members when they need assistance

## Festival-Specific Rules

### Commits

- **Every commit during festival execution uses `fest commit` only.** Never use
  raw `git commit`, `camp commit`, or `camp p commit` mid-festival; if
  `fest commit` does not auto-stage a path, stage it manually with `git add`
  first, then run `fest commit`.
- Commit messages are descriptive and never carry "Co-authored-by" tags.

### Working Location

- All code work happens in the linked worktree
  `projects/worktrees/camp/camp-hardening` (branch `camp-hardening`, cut from
  origin/main commit bc2ad1f). Do not work in `projects/camp` main checkout.
- This planning directory holds festival documents only; no code lives here.

### Build Profiles and Gates

- **Both build profiles must pass gates before any sequence completes:** run
  `just build`, `just test`, and `just lint` in the stable profile AND with the
  dev profile (`BUILD_TAGS=dev` / `CLI_PROFILE=dev`). TEST-1 proved failures
  hide in either profile; verifying one is not verifying both.
- `go vet -tags=integration ./...` must be clean once R14 lands it in lint.

### Local Enforcement Only (No GitHub Actions)

- GitHub CI is intentionally paused for cost. **Do not add or modify any
  GitHub Actions workflows.** All gate enforcement built in this festival is
  local: just recipes and pre-push hooks. Any finding or recommendation that
  suggests hosted CI (for example R13's "minimal CI" step) is implemented as
  local just recipes plus a pre-push hook instead.

### Testing Conventions

- Unit tests that mutate the filesystem follow the repo's existing host
  `t.TempDir` convention. Do not introduce a new isolation scheme for unit
  tests.
- Integration tests of real git/filesystem mutation behavior, including every
  destructive-path fix (worktrees clean, cp, fresh/prune, project remove, sync
  fallback), go in the container harness under `tests/integration/`
  (testcontainers, `//go:build integration`).
- Every behavioral fix ships with a regression test; error cases first.

### Source of Truth

- The authoritative work inventory is
  `001_INGEST/input_specs/recommendations.md` (R1-R36) and
  `001_INGEST/input_specs/second-pass-fable.md` (N-1 through N-27 plus the
  section 4 reordering). Re-confirm cited file:line locations against the
  worktree's branch point bc2ad1f before editing.
- Do not re-litigate the second pass's "hunted and found clean" list.
- camp uses its established error framework; never introduce new `fmt.Errorf`
  call sites (R24 ratchets the existing ones down).

---

## Compliance Checklist

Use this checklist for every task:

### Pre-Task

- [ ] Read FESTIVAL_RULES.md completely
- [ ] Understand task requirements
- [ ] Review existing code and patterns
- [ ] Plan approach

### During Task

- [ ] Follow coding standards
- [ ] Write tests alongside code
- [ ] Make small, focused commits
- [ ] Run tests frequently

### Task Completion

- [ ] All tests pass
- [ ] Code reviewed (self)
- [ ] Documentation updated
- [ ] Linters/formatters run
- [ ] Security considered
- [ ] Performance assessed
- [ ] Rules compliance verified

## Exceptions

If you need to deviate from these rules, document:

1. Which rule you're breaking
2. Why it's necessary
3. What the trade-offs are
4. Approval from festival planner (if required)