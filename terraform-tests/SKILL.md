---
name: terraform-tests
disable-model-invocation: true
description: >
  Use this skill when the user wants to write, generate, or create Terraform tests for a module
  or configuration. Trigger on phrases like "write terraform tests," "generate tests for this
  module," "add tftest files," "test my terraform," "create .tftest.hcl," "unit test my
  infrastructure," or any request to create automated tests for Terraform code.
version: latest
category: infrastructure
tags:
  - terraform
  - testing
  - infrastructure-as-code
  - tftest
  - hcl
---

# Terraform Tests

You generate `.tftest.hcl` test files for Terraform modules and configurations using the native
Terraform testing framework (Terraform ≥ 1.6). You explore the repo, understand what's being
built, and write tests that give real coverage — not boilerplate.

---

## Parallel Execution

Maximize use of the Agent tool. Whenever you have 2+ independent tasks, launch parallel agents.

- Module exploration: separate agents for variables, outputs, resources, providers
- Multiple modules: one agent per module directory
- Run exploration and research concurrently; synthesize before writing

Do not serialize what can be parallelized.

---

## Step 1 — Explore the Repo

Before writing any tests, understand what you're testing. Use parallel agents to:

1. **Discover structure** — find all `.tf` files, module directories, `variables.tf`, `outputs.tf`
2. **Identify resources** — what resource types are declared and what are their key attributes
3. **Read variable definitions** — types, defaults, validation rules, required vs optional
4. **Read output definitions** — what the module exposes and what callers depend on
5. **Check for existing tests** — find any `.tftest.hcl` files already present; don't duplicate

Key files to always read:
- `variables.tf` — inputs the tests must supply
- `outputs.tf` — values to assert against
- `main.tf` / `*.tf` — resources to understand expected behavior
- `versions.tf` / `required_providers` blocks — which providers are in use

---

## Step 2 — Classify Tests to Write

For each module, decide which test types apply:

### Unit Tests (mock providers)
Use `mock_provider` to avoid real API calls. Best for:
- Input validation (variable constraints, preconditions)
- Output value computation
- Resource count and naming logic
- Conditional resource creation

```hcl
mock_provider "aws" {}
```

### Contract Tests (known values from plan)
Use `command = plan` with `plan_only = true` assertions. Best for:
- Verifying a resource will be created/destroyed
- Confirming attribute values before apply
- Tagging and naming conventions

```hcl
run "verify_tags" {
  command = plan

  assert {
    condition     = aws_s3_bucket.main.tags["Environment"] == var.environment
    error_message = "Bucket must have the Environment tag set to var.environment"
  }
}
```

### Integration Tests (real apply)
Use `command = apply` (the default) for end-to-end validation. Best for:
- Outputs that depend on computed values (ARNs, IDs)
- Cross-resource dependencies
- State-dependent behavior

Only generate integration tests if the user explicitly requests them or the module cannot be
meaningfully tested without apply. Always note that integration tests incur real cloud costs.

---

## Step 3 — Write the Test Files

### File placement
- Put tests in a `tests/` subdirectory of the module, or in the module root if that's the
  project convention. Check what already exists before deciding.
- One `.tftest.hcl` file per logical concern (e.g., `unit_validation.tftest.hcl`,
  `defaults.tftest.hcl`, `naming.tftest.hcl`).

### Test file anatomy

```hcl
# tests/unit_validation.tftest.hcl

# Override providers to avoid real API calls (unit tests only)
mock_provider "aws" {
  alias = "mock"
}

# Shared variables used across multiple run blocks
variables {
  region      = "us-east-1"
  environment = "test"
}

# Each run block is an independent test case
run "valid_input_creates_bucket" {
  command = plan

  variables {
    bucket_name = "my-test-bucket"
  }

  assert {
    condition     = aws_s3_bucket.main.bucket == "my-test-bucket"
    error_message = "Bucket name did not match the input variable"
  }
}

run "empty_name_fails_validation" {
  command = plan

  variables {
    bucket_name = ""
  }

  expect_failures = [var.bucket_name]
}
```

### Assertion quality rules

1. **Assert the contract, not the implementation.** Test that outputs equal expected values,
   not that internal resource IDs have a specific format (unless that's the module's contract).

2. **Use `expect_failures` for validation rules.** When a variable has a `validation` block or
   a resource has a `precondition`/`postcondition`, write a test that triggers it.

3. **Cover defaults.** Write one `run` block with zero optional variables to verify the module
   works with only required inputs.

4. **Cover edge cases.** Empty strings, maximum lengths, boundary values for numeric variables,
   boolean toggles that enable/disable resources.

5. **Name `run` blocks descriptively.** The name should read like a test description:
   `"bucket_is_private_by_default"` not `"test1"`.

6. **One assertion per concern.** Don't put 10 unrelated assertions in one `run` block.
   Split them so failures are easy to diagnose.

---

## Step 4 — Explain the Tests

After writing the files, provide a summary:

```
TESTS WRITTEN
─────────────
Files created:
  tests/unit_validation.tftest.hcl    — input validation and defaults
  tests/naming_convention.tftest.hcl  — resource naming and tagging

Run with:
  terraform test                      # all tests
  terraform test -filter=tests/unit_validation.tftest.hcl  # one file

Test coverage:
  ✓ Required variables: [list]
  ✓ Optional variables with non-default values: [list]
  ✓ Validation rules tested: [list]
  ✓ Outputs verified: [list]
  ⚠ Not covered (requires apply / real credentials): [list]

Notes:
  [anything the user needs to know — missing provider config, assumed Terraform version, etc.]
```

---

## Rules That Override Your Defaults

1. **Read before writing.** Never generate a variable name or resource reference without reading
   the actual `.tf` files first. Wrong variable names cause immediate failures and erode trust.

2. **Prefer `command = plan` and `mock_provider`.** Integration tests with real `apply` have
   cost and side effects. Default to plan-only unless the test is impossible without apply.

3. **Never use placeholder values.** Every value in a test must be plausible for the resource
   type. Use realistic region names, valid CIDR blocks, real-looking ARN patterns, etc.

4. **Check Terraform version.** If `versions.tf` requires Terraform < 1.6, native tests aren't
   available. Flag this and ask the user whether to upgrade the constraint or use an alternative
   (e.g., Terratest in Go). Do not generate `.tftest.hcl` files for incompatible versions.

5. **Don't overwrite existing tests.** Read existing `.tftest.hcl` files first. Add new `run`
   blocks or new files; don't replace working tests.

6. **One file per concern, not one file per resource.** Group tests by what they're validating
   (validation, defaults, naming) rather than by resource type.

7. **Flag missing provider config.** If a module requires provider configuration that can't
   be mocked (e.g., it reads from a real state backend), note this clearly in the summary.

---

## HCL Reference

### `mock_provider` block
```hcl
mock_provider "<provider_name>" {
  # Optional: override specific data source results
  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "123456789012"
    }
  }
}
```

### `variables` block (top-level — shared across all runs)
```hcl
variables {
  region = "us-east-1"
}
```

### `run` block
```hcl
run "<descriptive_name>" {
  command = plan  # or "apply" (default)

  variables {
    # Override or extend top-level variables
  }

  # For testing that validation/preconditions fire:
  expect_failures = [
    var.some_variable,
    aws_resource.name,
  ]

  assert {
    condition     = <expression that must be true>
    error_message = "<what failed and why>"
  }
}
```

### `module` block (for testing a child module)
```hcl
run "test_child_module" {
  module {
    source = "./modules/networking"
  }

  assert {
    condition     = module.networking.vpc_id != ""
    error_message = "VPC ID should not be empty"
  }
}
```
