# 🏗️ Central CI/CD Workflow Repository

**Centralized source of truth for all CI/CD pipelines** used within the organization.  
This repository provides a library of **reusable, secure, and standardized workflows and actions** for building, testing, scanning, and deploying applications.

> 🎯 **Goal:** Enforce governance, security, and maintainability by abstracting CI/CD logic away from individual application repositories.

---

## 🧩 Architecture Overview

This repository implements a **Stage-based Fan-out architecture**.

### 🧱 Components

| Component | Description |
|------------|--------------|
| **Application Repo** | Defines a single workflow file (`.github/workflows/main-ci.yml`). |
| **Orchestrator** | Central `orchestrator.yml` workflow that routes based on profile. |
| **Routing** | Reads `pipeline_profile` input (e.g., `java-openshift`) to define stages. |
| **Fan-out** | Each stage (build, test, scan, etc.) runs the correct job conditionally. |
| **Reusable Workflows** | Versioned YAML files with specific logic (e.g., `1-build-java.yml`). |
| **Composite Actions** | Setup actions (e.g., `setup-java-maven`) used by workflows. |

---

### 🔄 Visual Flow

```
[App Repo: ci.yml]
      |
      V
[Orchestrator: orchestrator.yml] (Receives 'pipeline_profile')
      |
      +--- (if: 'java') --> [build-java] --> [test-java] --> [scan-java] --+
      |                                                                 |
      +--- (if: 'dotnet') -> [build-dotnet] -> [test-dotnet] -> [scan-dotnet] --+
      |                                                                 |
      +--- (if: 'android') -> [build-android] -> [test-android] -> [scan-android] --+
                                                                          |
                                                                          V
                                                                    [sign-artifact]
                                                                          |
                                                                          V
                                                                    [deploy-...]
```

---

## 🚀 How to Use (For Application Developers)

Create a single workflow in your app repository that **delegates** to the central orchestrator.

**File:** `your-app-repo/.github/workflows/main-ci.yml`

```yaml
name: Application CI/CD

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  call-orchestrator:
    name: Call Central CI Orchestrator

    permissions:
      contents: read
      id-token: write
      packages: write
      security-events: write

    uses: tu-organizacion/central-cicd-repo/.github/workflows/orchestrator.yml@v1.0.0
    
    with:
      pipeline_profile: 'java-openshift'
      sonar-project-key: 'my-app-sonar-key'
      snyk-org-slug: 'my-snyk-org'

    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      OPENSHIFT_TOKEN: ${{ secrets.OPENSHIFT_TOKEN_FOR_DEPLOY }}
```

---

## 🗂️ Repository Structure

```
central-cicd-repo/
├── .github/
│   ├── actions/
│   │   └── setup-java-maven/ (Composite Actions)
│   └── workflows/
│       ├── orchestrator.yml
│       ├── 1-build-java.yml
│       ├── 2-test-java.yml
│       ├── 3-scan-java.yml
│       ├── 4-sign-artifact.yml
│       ├── 5-deploy-java-openshift.yml
│       └── ...
└── README.md
```

**Descriptions:**
- **actions/** → Tool setup composite actions.
- **workflows/** → Reusable workflows.
- **orchestrator.yml** → Main entry point (“brain”).
- **1-build-*.yml**, **2-test-*.yml**, etc. → Stage definitions per technology.

---

## 🧮 Available Pipeline Profiles (`pipeline_profile`)

| Profile | Description |
|----------|--------------|
| `java-openshift` | Java app deployed to OpenShift |
| `java-middlewares` | Java middleware services |
| `.Net` | .NET applications |
| `android` | Android builds |
| `ios` | iOS apps |
| `javascript` | Node.js / JS apps |
| `sql` / `nosql` | Database pipelines |
| `vendor-binary` | Vendor binary deployments |
| `middleware: nginx` | NGINX middleware |
| `kafka` | Kafka microservices |
| ... | (Add new profiles as needed) |

---

## 🧑‍💻 How to Contribute (For Platform Team)

To add a new pipeline (e.g., **python-django**):

1. **Create Composite Actions**
   - `actions/setup-python/action.yml` for Python setup (pip, cache, etc.)

2. **Add Reusable Workflows**
   - `1-build-python.yml`
   - `2-test-python.yml`
   - `3-scan-python.yml`
   - `5-deploy-python-django.yml`

3. **Update Orchestrator**
   - Add jobs: `build-python`, `test-python`, etc.
   - Include conditions:
     ```yaml
     if: inputs.pipeline_profile == 'python-django'
     ```
   - Define dependencies using `needs:`.

4. **Documentation**
   - Add the new profile to this README.

5. **Versioning**
   - Create a new tag (e.g., `v1.1.0`).

---

## 🏷️ Versioning Best Practices

**❌ Don’t use unstable references:**
```yaml
uses: tu-organizacion/central-cicd-repo/.github/workflows/orchestrator.yml@main
```

**✅ Use stable tags:**
```yaml
uses: tu-organizacion/central-cicd-repo/.github/workflows/orchestrator.yml@v1.0.0
```

> This ensures consistent builds and controlled upgrades.

---

## 🧠 Summary

- Centralized CI/CD logic for consistency and governance.  
- Reusable, secure workflows per technology.  
- Versioned orchestration for reliable delivery.  
- Extensible and easy to maintain.
