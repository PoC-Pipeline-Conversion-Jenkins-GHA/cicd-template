Central CI/CD Workflow Repository

This repository is the centralized source of truth for all CI/CD pipelines used within the organization. It provides a library of reusable, secure, and standardized workflows and actions for building, testing, scanning, and deploying applications.

The primary goal of this repository is to enforce governance, security, and maintainability by abstracting CI/CD logic away from individual application repositories.

Architecture Overview

This repository uses a "Stage-based Fan-out" architecture.

Application Repo: An application (e.g., my-java-app) defines a single .github/workflows/main-ci.yml file.

Orchestrator: The application calls the main orchestrator.yml workflow in this repository.

Routing: The orchestrator.yml reads the pipeline_profile input (e.g., java-openshift) and defines the stages (build, test, scan, sign, deploy) using needs: dependencies.

Fan-out: Each stage (e.g., build) is a collection of jobs (e.g., build-java, build-dotnet). An if: condition activates the correct job based on the profile.

Reusable Workflows: Each job (e.g., build-java) calls a specific, versioned reusable workflow (e.g., 1-build-java.yml) that contains the actual logic for that stage and technology.

Composite Actions: The reusable workflows use actions/ (e.g., setup-java-maven) to configure the runner environment.

Visual Flow:

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


How to Use (For Application Developers)

Do not add complex logic to your application repository. Instead, create a single workflow file that calls this repository's orchestrator.

File: your-app-repo/.github/workflows/main-ci.yml

name: Application CI/CD

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  call-orchestrator:
    name: Call Central CI Orchestrator
    
    # Permissions required to delegate to the reusable workflows
    permissions:
      contents: read
      id-token: write      # For OIDC authentication (e.g., Sign, Deploy)
      packages: write      # For publishing artifacts
      security-events: write # For uploading scan results

    # 1. CALL THE ORCHESTRATOR
    #    Replace 'tu-organizacion' and use a version tag (e.g., @v1.0.0)
    uses: tu-organizacion/central-cicd-repo/.github/workflows/orchestrator.yml@main
    
    # 2. PROVIDE INPUTS (with:)
    #    Define WHAT pipeline to run and provide its parameters.
    with:
      # --- REQUIRED ---
      # See 'Available Pipeline Profiles' below
      pipeline_profile: 'java-openshift' 
      
      # --- Parameters for this profile ---
      sonar-project-key: 'my-app-sonar-key'
      snyk-org-slug: 'my-snyk-org'
      # (Add other inputs as required by the profile)

    # 3. PASS SECRETS
    #    Pass only the secrets required by your chosen profile.
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      OPENSHIFT_TOKEN: ${{ secrets.OPENSHIFT_TOKEN_FOR_DEPLOY }}


Repository Structure

This repository is organized into two main components:

central-cicd-repo/
├── .github/
│   │
│   ├── actions/
│   │   # ... (Composite Actions: "Reusable tool setups")
│   │
│   └── workflows/
│       │
│       ├── orchestrator.yml    # (Main entry point. Defines stages and routes.)
│       │
│       # --- Reusable Workflows (Flattened by stage) ---
│       ├── 1-build-java.yml
│       ├── 2-test-java.yml
│       ├── 3-scan-java.yml
│       ├── 4-sign-artifact.yml
│       ├── 5-deploy-java-openshift.yml
│       └── ... (etc.)
│
└── README.md


actions/: Contains Composite Actions. These are "tool" configurations (e.g., setup-java-maven). They prepare the runner but do not execute major logic.

workflows/: Contains Reusable Workflows.

orchestrator.yml: The "brain" or "router". It is the only workflow intended to be called by external application repositories.

1-build-*.yml, 2-test-*.yml, etc.: These are the "processes" or "stages". They are called by the orchestrator and define the logic for a specific stage (e.g., "build Java") by using the actions/ tools.

Available Pipeline Profiles (pipeline_profile)

This is the "menu" of available end-to-end pipelines. Pass one of these values in your main-ci.yml:

java-openshift

java-middlewares

.Net

Android

iOS

javascript

sql / no SQL

Vendor binary

middleware: Nginx

Kafka

(...add other profiles as they are built...)

How to Contribute (For Platform Team)

To add a new pipeline (e.g., python-django):

Actions: Create a new actions/setup-python/action.yml (if it doesn't exist) to install Python, pip, and cache dependencies.

Reusable Workflows: Create the new stage workflows in the workflows/ directory:

1-build-python.yml

2-test-python.yml

3-scan-python.yml (e.g., using Snyk, Bandit)

5-deploy-python-django.yml

Orchestrator: Edit orchestrator.yml to add the new logic:

Add new build-python, test-python, scan-python, deploy-python-django jobs.

Add the needs: dependencies for each new job.

Add the if: inputs.pipeline_profile == 'python-django' condition to each new job.

Ensure the uses: path points to the new workflow files (e.t., uses: ./.github/workflows/1-build-python.yml).

Documentation: Add python-django to the "Available Pipeline Profiles" list in this README.md.

Versioning: Create a new version tag (e.g., v1.1.0) for the repository so applications can start using the new profile.

Versioning Best Practices

Application repositories SHOULD NOT call the orchestrator using @main. This is unstable and can break builds during development.

DO NOT DO THIS:
uses: tu-organizacion/central-cicd-repo/.github/workflows/orchestrator.yml@main

DO THIS:
Use a stable version tag (e.g., @v1.0.0). This ensures that application builds are stable and only update their CI/CD process when they are ready.
uses: tu-organizacion/central-cicd-repo/.github/workflows/orchestrator.yml@v1.0.0