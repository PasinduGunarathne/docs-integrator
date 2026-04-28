---
title: Connecting a Runtime to ICP
---

# Connecting a Runtime to ICP

Both WSO2 Integrator (BI) and WSO2 Micro Integrator (MI) runtimes connect to the ICP Server on port **`9445`** (HTTPS) to register themselves and send periodic heartbeats. Ensure this port is reachable from all integration nodes.

---

## WSO2 Integrator (BI) Runtime

Add the following to the integration's `Config.toml`:

```toml
[wso2.icp.runtime.bridge]
environment = "dev"
project = "<project-name>"
integration = "<integration-name>"
runtime = "<unique-runtime-id>"
secret = "<shared-secret>"
# serverUrl = "https://<icp-host>:9445"
```

Enable remote management in `Ballerina.toml`:

```toml
[build-options]
remoteManagement = true
```

Import the ICP runtime bridge in the integration entrypoint (`main.bal`):

```ballerina
import wso2/icp.runtime.bridge as _;
```

### BI Configuration Keys

| Key           | Description                                                                                |
| ------------- | ------------------------------------------------------------------------------------------ |
| `environment` | Logical environment handle (e.g. `dev`, `staging`, `prod`)                                  |
| `project`     | Project handle shown in the ICP dashboard                                                    |
| `integration` | Integration handle shown in the ICP dashboard                                                |
| `runtime`     | Unique identifier for this runtime instance                                                |
| `secret`      | Shared secret used to authenticate the runtime with the ICP Server                         |
| `serverUrl`   | ICP Server URL for runtime communication — defaults to `https://localhost:9445` if not set |

---

## WSO2 Micro Integrator (MI) Runtime

Add the following to the MI server's `deployment.toml`:

```toml
[icp_config]
enabled = true
environment = "dev"
project = "<project-name>"
integration = "<integration-name>"
runtime = "<unique-runtime-id>"
secret = "<shared-secret>"
# icp_url = "https://<icp-host>:9445"
```

### MI Configuration Keys

| Key           | Description                                                                                |
| ------------- | ------------------------------------------------------------------------------------------ |
| `enabled`     | Set to `true` to activate ICP connectivity                                                 |
| `environment` | Logical environment handle (e.g. `dev`, `staging`, `prod`)                                  |
| `project`     | Project handle shown in the ICP dashboard                                                    |
| `integration` | Integration handle shown in the ICP dashboard                                                |
| `runtime`     | Unique identifier for this runtime instance                                                |
| `secret`      | Shared secret used to authenticate the runtime with the ICP Server                         |
| `icp_url`     | ICP Server URL for runtime communication — defaults to `https://localhost:9445` if not set |
