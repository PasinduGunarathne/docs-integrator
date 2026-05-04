---
title: Configurations
---

# Configurations

Configuration artifacts externalize values that change between environments using Ballerina's `configurable` keyword. This separates environment-specific settings (URLs, credentials, feature flags) from your integration logic. At runtime, the integration resolves each variable from `Config.toml`, environment variables, or command-line arguments.

For the underlying configuration model, the full list of supported sources, and resolution priority, see [Configuration management](../../design-logic/configuration-management.md).

## Adding a configuration

1. Open your integration project in **WSO2 Integrator**.

   ![WSO2 Integrator sidebar showing the project structure with Configurations listed](/img/develop/integration-artifacts/supporting/configurations/step-1.png)

2. Click **+** next to **Configurations** in the sidebar.

3. In the **Add Configurable Variable** panel, fill in the following fields:

   ![Add Configurable Variable form showing Variable Name, Variable Type, Default Value, and Documentation fields](/img/develop/integration-artifacts/supporting/configurations/step-2.png)

   | Field | Description |
   |---|---|
   | **Variable Name** | The identifier used to reference the variable in code (for example, `apiEndpoint`). Required. |
   | **Variable Type** | The Ballerina type of the variable (for example, `string`, `int`, `boolean`, or a record type). Required. |
   | **Default Value** | An optional default value. Leave empty to make the variable required — the integration fails to start unless a value is supplied through `Config.toml`, an environment variable, or a CLI argument. |
   | **Documentation** | Optional Markdown description rendered as inline documentation. |

4. Click **Save**. The variable is added to your project's configurable declarations.

Declare configurable variables at the module level using the `configurable` keyword:

```ballerina
// config.bal

// Required configuration -- must be supplied at runtime
configurable string apiEndpoint = ?;
configurable string apiKey = ?;

// Optional configuration with defaults
configurable int maxRetries = 3;
configurable decimal timeoutSeconds = 30.0d;
configurable boolean enableCache = true;
configurable int cacheMaxSize = 1000;

// Grouped configuration using a record type
type NotificationConfig record {|
    boolean emailEnabled;
    boolean slackEnabled;
    string slackWebhookUrl;
|};

configurable NotificationConfig notificationConfig = {
    emailEnabled: true,
    slackEnabled: false,
    slackWebhookUrl: ""
};
```

Variables initialized with `?` have no default and must be supplied at runtime; otherwise, the integration fails to start.

## Viewing configurations

Click the configurations icon next to **Configurations** in the sidebar to open the **Configurable Variables** panel.

![Configurable Variables panel showing variables grouped by Integration and Imported libraries](/img/develop/integration-artifacts/supporting/configurations/step-3.png)

The panel organizes variables into two groups:

- **Integration** — variables declared in your integration project. Each entry shows the variable name, type, and default value.
- **Imported libraries** — configurable variables exposed by libraries your integration depends on (for example, `ballerina/http` or `ballerina/log`).

Use the **Search Configurables** box to filter by name. Click a variable to edit or delete it.

## Providing values

Place a `Config.toml` file at the project root (alongside `Ballerina.toml`) to supply values for configurable variables. The runtime reads it automatically at startup.

```toml
apiEndpoint = "https://api.example.com/v2"
apiKey = "sk-abc123"
maxRetries = 5
timeoutSeconds = 60.0
enableCache = true
cacheMaxSize = 5000

[notificationConfig]
emailEnabled = true
slackEnabled = true
slackWebhookUrl = "https://hooks.slack.com/services/..."
```

For the full list of supported types, alternative value sources (environment variables, CLI arguments, inline TOML), and resolution priority, see [Configuration management](../../design-logic/configuration-management.md). To target different environments, point `BAL_CONFIG_FILES` at a per-environment file — see [Per-environment configuration](../../design-logic/configuration-management.md#per-environment-configuration).

## Best practices

| Practice | Description |
|---|---|
| **Dedicated file** | Keep configurable declarations in a dedicated `config.bal` file so all settings are easy to locate. |
| **Mark required values explicitly** | Use `?` for values that must come from the environment (endpoints, credentials) so misconfiguration fails fast at startup. |
| **Group related settings** | Use record types to group settings that belong to the same subsystem (for example, database, CRM, notifications). |
| **Never commit secrets** | Keep secrets out of `Config.toml` files in version control. Supply them through environment variables or a gitignored secrets file. See [Secrets and encryption](../../../deploy-operate/secure/secrets-encryption.md). |
| **Document defaults** | Use the **Documentation** field (or code comments) to explain the purpose and valid range of each setting. |

## What's next

- [Configuration management](../../design-logic/configuration-management.md) — Deeper reference for configuration sources, priority order, and module-qualified keys.
- [Secrets and encryption](../../../deploy-operate/secure/secrets-encryption.md) — Securely manage credentials and other sensitive values.
- [Connections](connections.md) — Use configurable variables to parameterize connections.
