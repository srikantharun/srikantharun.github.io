# GitLab CI/CD Components: Build Once, Use Everywhere

**A Practical Guide to Creating, Publishing, and Consuming Reusable Pipeline Components**

---

## Overview

GitLab CI/CD Components are reusable, versioned building blocks for pipelines. Instead of copy-pasting YAML across projects, you create a component once, publish it to the CI/CD Catalog, and consume it anywhere.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COMPONENT LIFECYCLE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│   │   DEVELOP    │────▶│   PUBLISH    │────▶│   CONSUME    │        │
│   │  Component   │     │  to Catalog  │     │  in Projects │        │
│   └──────────────┘     └──────────────┘     └──────────────┘        │
│         │                    │                    │                  │
│         ▼                    ▼                    ▼                  │
│   templates/           CI/CD Catalog        include:                 │
│   └── my-job.yml       (Registry)           - component: ...        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Why Components Over Templates?

| Aspect | Components | Templates |
|--------|-----------|-----------|
| **Versioning** | Semantic versioning (v1.2.3) | Branch/tag includes |
| **Discovery** | Searchable CI/CD Catalog | Manual documentation |
| **Inputs** | Typed, validated `spec:inputs` | Unstructured variables |
| **Scope** | Single-purpose, composable | Often entire pipelines |
| **Testing** | First-class testing support | Ad-hoc |

---

## The Complete Workflow

### Step 1: Create Component Project Structure

```
my-components/
├── templates/                    # REQUIRED: All components live here
│   ├── python-test.yml          # Single-file component
│   ├── docker-build.yml         # Another component
│   └── deploy/                   # Multi-file component
│       ├── template.yml         # Main component file
│       └── scripts/
│           └── deploy.sh
├── tests/                        # Component tests
│   ├── test-python-test.yml
│   └── test-docker-build.yml
├── .gitlab-ci.yml               # Pipeline to test & release
└── README.md                    # Documentation
```

### Step 2: Write a Component

```yaml
# templates/python-test.yml

# 1. SPEC SECTION - Define inputs with types and defaults
spec:
  inputs:
    stage:
      description: "Pipeline stage"
      default: test
    python_version:
      description: "Python version to use"
      default: "3.11"
      options: ["3.9", "3.10", "3.11", "3.12"]
    coverage_threshold:
      description: "Minimum coverage percentage"
      type: number
      default: 80

---
# 2. JOB DEFINITION - Uses inputs via $[[ inputs.name ]]

python-test:
  stage: $[[ inputs.stage ]]
  image: python:$[[ inputs.python_version ]]-slim
  before_script:
    - pip install pytest pytest-cov
  script:
    - pytest --cov=src --cov-fail-under=$[[ inputs.coverage_threshold ]]
  coverage: '/TOTAL.*\s+(\d+%)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

### Step 3: Test the Component

```yaml
# tests/test-python-test.yml

include:
  # Reference component from current commit (for testing)
  - component: $CI_SERVER_FQDN/$CI_PROJECT_PATH/python-test@$CI_COMMIT_SHA
    inputs:
      python_version: "3.11"
      coverage_threshold: 50

stages:
  - test
  - verify

# Component adds 'python-test' job automatically

verify-component:
  stage: verify
  needs: [python-test]
  script:
    - echo "Component test passed!"
```

### Step 4: CI/CD Pipeline for Publishing

```yaml
# .gitlab-ci.yml

stages:
  - validate
  - test
  - release

# Validate component structure
validate:
  stage: validate
  image: alpine:latest
  script:
    - |
      for file in templates/*.yml templates/*/template.yml; do
        if [ -f "$file" ]; then
          grep -q "^spec:" "$file" || (echo "ERROR: $file missing spec:" && exit 1)
          echo "PASS: $file"
        fi
      done

# Test each component
test-python-component:
  stage: test
  trigger:
    include:
      - local: tests/test-python-test.yml
    strategy: depend

# Release to catalog on tag
release:
  stage: release
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
  script:
    - echo "Releasing $CI_COMMIT_TAG"
  release:
    tag_name: $CI_COMMIT_TAG
    name: "Release $CI_COMMIT_TAG"
    description: "CI/CD Components Release"
```

### Step 5: Enable CI/CD Catalog

1. Go to **Settings > General**
2. Expand **Visibility, project features, permissions**
3. Enable **CI/CD Catalog project**

### Step 6: Publish with Semantic Version Tag

```bash
git tag -a v1.0.0 -m "Initial release"
git push origin v1.0.0
```

The pipeline runs, tests pass, and the component appears in the CI/CD Catalog.

---

## Consuming Components

### Basic Usage

```yaml
# Consumer project's .gitlab-ci.yml

include:
  - component: gitlab.example.com/devops/components/python-test@1.0.0
    inputs:
      python_version: "3.11"
      coverage_threshold: 85

stages:
  - test
  - build
  - deploy

# python-test job is automatically added!

build:
  stage: build
  script:
    - make build
```

### Multiple Components

```yaml
include:
  # Linting
  - component: gitlab.example.com/components/python-lint@1.0.0
    inputs:
      lint_paths: "src/"

  # Testing
  - component: gitlab.example.com/components/python-test@1.0.0
    inputs:
      coverage_threshold: 90

  # Security
  - component: gitlab.example.com/components/secret-scan@1.2.0

  # Docker
  - component: gitlab.example.com/components/docker-build@2.0.0
    inputs:
      push: true

stages:
  - validate
  - test
  - build
  - deploy

# All component jobs are injected automatically!

deploy:
  stage: deploy
  script:
    - ./deploy.sh
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### Version Pinning Strategies

```yaml
include:
  # Exact version (recommended for production)
  - component: gitlab.example.com/components/pytest@1.2.3

  # Latest 1.x (auto-updates for patches)
  - component: gitlab.example.com/components/lint@~1

  # Latest (not recommended - may break)
  - component: gitlab.example.com/components/scan@~latest
```

---

## Component Registry Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         COMPONENT AUTHOR                                 │
│                                                                          │
│   1. Create component          2. Push & Tag              3. Published   │
│   ┌─────────────────┐         ┌─────────────────┐       ┌─────────────┐ │
│   │ templates/      │         │ git tag v1.0.0  │       │  CI/CD      │ │
│   │ └── job.yml     │────────▶│ git push --tags │──────▶│  Catalog    │ │
│   │                 │         │                 │       │  Registry   │ │
│   │ spec:           │         │ Pipeline runs:  │       │             │ │
│   │   inputs:       │         │ - validate      │       │ v1.0.0 ✓    │ │
│   │     version:    │         │ - test          │       │ v1.1.0 ✓    │ │
│   │       default:  │         │ - release       │       │ v2.0.0 ✓    │ │
│   └─────────────────┘         └─────────────────┘       └─────────────┘ │
│                                                                │         │
└────────────────────────────────────────────────────────────────┼─────────┘
                                                                 │
                                                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         COMPONENT CONSUMER                               │
│                                                                          │
│   4. Include in pipeline       5. Pipeline executes      6. Jobs run    │
│   ┌─────────────────┐         ┌─────────────────┐       ┌─────────────┐ │
│   │ .gitlab-ci.yml  │         │ GitLab fetches  │       │ python-test │ │
│   │                 │         │ component from  │       │ docker-build│ │
│   │ include:        │────────▶│ catalog at      │──────▶│ deploy      │ │
│   │ - component:    │         │ specified       │       │             │ │
│   │     .../job@1.0 │         │ version         │       │ All jobs    │ │
│   │   inputs:       │         │                 │       │ injected!   │ │
│   │     key: value  │         │                 │       │             │ │
│   └─────────────────┘         └─────────────────┘       └─────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Input Types and Validation

Components support typed inputs with validation:

```yaml
spec:
  inputs:
    # String with options (dropdown in UI)
    environment:
      type: string
      default: "staging"
      options: ["dev", "staging", "production"]

    # Boolean toggle
    dry_run:
      type: boolean
      default: false

    # Number with validation
    replicas:
      type: number
      default: 3

    # Array of values
    target_platforms:
      type: array
      default: ["linux/amd64", "linux/arm64"]

    # Regex validation
    version_tag:
      type: string
      regex: "^v\\d+\\.\\d+\\.\\d+$"
```

---

## Real-World Example: Complete Python Pipeline

### Component Project Structure

```
python-components/
├── templates/
│   ├── lint.yml           # Ruff linting
│   ├── test.yml           # Pytest with coverage
│   ├── security-scan.yml  # Bandit + safety
│   └── publish.yml        # PyPI publishing
├── tests/
│   └── ...
└── .gitlab-ci.yml
```

### Consumer Gets Full Pipeline

```yaml
# Any Python project's .gitlab-ci.yml

include:
  - component: gitlab.com/my-org/python-components/lint@1.0.0
  - component: gitlab.com/my-org/python-components/test@1.0.0
    inputs:
      coverage_threshold: 85
  - component: gitlab.com/my-org/python-components/security-scan@1.0.0
  - component: gitlab.com/my-org/python-components/publish@1.0.0
    inputs:
      pypi_repository: "https://pypi.org/simple/"

stages:
  - validate
  - test
  - security
  - publish

# That's it! Full pipeline with 4 lines of config.
```

---

## Curated Components Repository

For a complete collection of production-ready components covering Python, Java, Node.js, Terraform, Docker, security scanning, and cloud deployments, see:

**[gitlab-curated-components](https://github.com/srikantharun/gitlab-curated-components)**

---

## Key Takeaways

1. **Components are versioned** - Pin versions for stability, use `~1` for auto-updates
2. **Single responsibility** - One component = one job/task
3. **Smart defaults** - Most inputs should have sensible defaults
4. **Test before publish** - Use child pipelines to validate components
5. **Semantic versioning** - Breaking changes = major version bump

---

## Resources

- [GitLab CI/CD Catalog](https://gitlab.com/explore/catalog)
- [Components Documentation](https://docs.gitlab.com/ci/components/)
- [Component Examples](https://docs.gitlab.com/ci/components/examples/)
- [to-be-continuous Components](https://gitlab.com/to-be-continuous)
