---
title: Migrate from MuleSoft
---

# Migrate from MuleSoft

## Overview

The MuleSoft migration tool converts MuleSoft Anypoint flows (XML configurations) to Ballerina code. It handles HTTP listeners, request connectors, DataWeave transformations, routers, and error handling patterns.

## Run the MuleSoft migration tool

The migration wizard guides you through a 5-step process to convert your MuleSoft project(s) into a WSO2 Integrator project.

### Step 1: Configure source

1. Open WSO2 Integrator, click **More Actions**, and select **Migrate Integrations from Other Vendors**.
2. Select **MuleSoft** as the source platform.
3. Under **Select a Project Folder or Directory**, click **Browse** and select your MuleSoft project directory or a directory containing multiple projects.
4. Under **Source Layout**, select one of the following:
   - **Single Project** — The source path points to a single project directory.
   - **Multiple Projects** — The source path points to a directory containing one or more project directories.

   > **Note:** The **Source Layout** section appears only after you select a directory.

5. Expand **Configure MuleSoft Settings** to set optional parameters:
   - **Force Version** — Select a specific Mule version or use **Auto Detect** to let the tool determine it from your project.
6. Click **Generate Report**.

   ![Configure source step](/img/develop/tools/migration-tools/mule-configure-source.png)

### Step 2: Report generation

The wizard performs a dry run against your source project(s) to generate a coverage report before the actual migration begins.

When the dry run completes, the wizard displays a summary of the migration coverage:

- **Migration Coverage** — Percentage of code lines that were automatically migrated.
- **Total code lines** — Total number of source code lines analyzed.
- **Migratable code lines** — Lines successfully converted to Ballerina.
- **Non-migratable code lines** — Lines that require manual attention.

   ![Report generation step](/img/develop/tools/migration-tools/mule-report-generation.png)

Click **View Full Report** to open the full HTML report. The report includes:

- **Migration Coverage Overview** — Overall coverage percentage with a breakdown of total, migratable, and non-migratable code lines.
- **Breakdown Components** — Separate coverage for Mule Elements and DataWeave expressions.
- **Manual Work Estimation** — Estimated effort (best, average, and worst case) for completing non-migratable items.
- **Currently Unsupported Elements** — List of elements that could not be automatically migrated.
- **Element Blocks that Require Manual Conversion** — Specific code blocks that need manual implementation.

   ![Full migration report](/img/develop/tools/migration-tools/mule-sample-migration-report.png)

Click **Save Report** to download the report for future reference.

Click **Configure Destination** to proceed, or **Done** to exit the wizard.

### Step 3: Configure destination

1. Enter an **Integration Name** for your migrated project.
2. Configure the project settings:
   - **Project Name** — Name of the project (defaults to `Default`).
   - **Create within a project** — Enable project mode to manage multiple integrations and libraries within a single repository.
   - **Select Path** — Choose where to create the migrated project.
3. Click **Start Migration**.

   ![Configure destination step](/img/develop/tools/migration-tools/mule-configure-destination.png)

### Step 4: Rule-based migration

The wizard runs the automated rule-based migration and displays progress in the migration log.

After the migration completes successfully, the **AI Enhancement (Recommended)** section appears. Select one of the following:

- **Enhance with AI** — AI automatically resolves unmapped elements, fixes build errors, and improves migration quality.
- **Skip for Now – Enhance Later** — Keep the project as-is. You can trigger AI enhancement later from the WSO2 Integrator Copilot.

Click **Start AI Enhancement** to proceed to Step 5, or if you chose to skip, click **Open Project** to open the migrated project or **Done** to exit.

   ![Rule-based migration step](/img/develop/tools/migration-tools/rule-based-migration.png)

### Step 5: AI enhancement

This step runs only if you selected **Enhance with AI** in Step 4.

The wizard first checks whether you are signed in. If not, a sign-in panel appears:

1. Click **Login using WSO2 Integration Platform** to sign in using SSO, or use one of the alternative options:
   - **Enter your Anthropic API key**
   - **Enter your AWS Bedrock credentials**
   - **Enter your Google Vertex AI credentials**
2. To skip AI enhancement and exit, click **Skip and Done**.

   ![Sign-in panel](/img/develop/tools/migration-tools/sign-in-for-ai-enhancement.png)

After signing in, the AI agent runs automatically and streams its progress. The agent resolves unmapped elements, fixes build errors, and improves the overall quality of the migrated code.

While the agent is running:

- Click **Pause** to pause the AI enhancement. Click **Resume** to continue.
- Click **Done** to exit the wizard, or **Open Project** to open the project without waiting for the agent to finish.

   ![Ehancing with ai-agent](/img/develop/tools/migration-tools/mule-ai-enhancement.png)

When the AI enhancement completes, the status shows **AI Enhancement completed**. Click **Open Project** to open the migrated project or **Done** to exit.

```bash
# Migrate a MuleSoft project
bal migrate mule -i /path/to/mule-project/ -o migrated/

# Specify the MuleSoft version
bal migrate mule -i /path/to/mule-project/ --version 4 -o migrated/

# Generate report only
bal migrate mule -i /path/to/mule-project/ --report-only
```

## Component mapping

| MuleSoft component | Ballerina equivalent |
|---|---|
| HTTP Listener | `http:Listener` + service |
| HTTP Request | `http:Client` |
| Database Connector | `mysql:Client` / `postgresql:Client` |
| Transform (DataWeave) | Ballerina query expressions + data mapper |
| Flow Reference | Function call |
| Choice Router | `if/else` or `match` |
| For Each | `foreach` |
| Scatter-Gather | Workers (parallel execution) |
| Try | `do/on fail` |
| Object Store | Configurable state management |
| Scheduler | `task:Listener` |

## Example: MuleSoft flow to Ballerina

**MuleSoft flow (XML):**
```xml
<flow name="getOrderFlow">
    <http:listener config-ref="HTTP_Listener" path="/orders/{orderId}" method="GET"/>
    <logger message="Received order request for #[attributes.uriParams.orderId]" level="INFO"/>
    <http:request config-ref="Backend_Request" method="GET"
                  path="/orders/#[attributes.uriParams.orderId]"/>
    <ee:transform>
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
                output application/json
                ---
                {
                    id: payload.orderId,
                    customer: payload.customerName,
                    total: payload.amount
                }]]>
            </ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>
```

**Generated Ballerina code:**
```ballerina
import ballerina/http;
import ballerina/log;

configurable string backendUrl = ?;

final http:Client backendClient = check new (backendUrl);

service /orders on new http:Listener(8090) {

    resource function get [string orderId]() returns json|error {
        log:printInfo("Received order request", orderId = orderId);

        json payload = check backendClient->get("/orders/" + orderId);

        // Transformed from DataWeave
        return {
            id: check payload.orderId,
            customer: check payload.customerName,
            total: check payload.amount
        };
    }
}
```
