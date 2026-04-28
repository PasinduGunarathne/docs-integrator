---
title: Create Your First Project
---

# Create Your First Project

Create a new WSO2 Integrator project and understand what gets generated.

## Create a project

1. Open VS Code
2. Open the Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`)
3. Select **WSO2 Integrator: Create New Project**
4. Choose a directory and project name
5. The extension generates the project structure

## Project structure

```
my-integration/
├── Ballerina.toml          # Project metadata and dependencies
├── Config.toml             # Environment-specific configuration
├── main.bal                # Main integration file
├── modules/                # Additional modules
├── tests/                  # Test files
└── resources/              # Static resources (schemas, configs)
```

## Run the project

1. Select the **Run** button in the VS Code toolbar, or
2. Open a terminal and run: `bal run`
3. You should see the service start successfully

## What's next

- [Understand the IDE](understand-the-ide.md) — Learn the visual designer layout
- [Build an API integration](quick-start-api.md) — Your first real integration
