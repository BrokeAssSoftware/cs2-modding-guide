# Architecture Overview

Use this section to build Vice & Order modules that follow consistent structure, initialise cleanly, and interoperate across the micro-mod ecosystem.

- [Module Layout](module-layout.md) - naming conventions, folder structure, and shared MSBuild properties.
- [Lifecycle and Initialization](lifecycle-and-initialization.md) - step-by-step load sequence plus sample `Mod` implementation.
- [Settings and Data Management](settings-and-data.md) - options UI patterns, localisation, logging, and data storage boundaries.
- [System Scheduling](system-scheduling.md) - disable vanilla systems, register custom systems, and manage update phases.
- [Dependency Strategy](dependency-strategy.md) - coordinate shared libraries, feature flags, and inter-mod hooks.
- [Performance Targets and Terminology](performance-and-terminology.md) - reference hardware, frame budgets, canonical vocabulary, and draft API contracts.

Start here when designing a new module or updating existing ones to match current architectural guidelines.
