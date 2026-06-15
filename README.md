

# 🚀 Power BI Tenant Monitoring & Governance using Admin REST APIs

Maintaining transparency, ensuring optimal performance, and enforcing governance across a large-scale Power BI tenant can be challenging. By utilizing the **Power BI Admin REST APIs**—specifically through metadata scanning and refresh history endpoints—you can programmatically extract granular metadata of every workspace, report, dataset, and data source. 

By dumping this API data into a centralized repository, you can build a comprehensive **Power BI Admin & Governance Report** that provides complete transparency over performance, access, failures, and structural metadata.

This repository outlines the architecture, prerequisites, and endpoints required to build this solution, utilizing strictly official Microsoft documentation.

![Power BI Admin Monitoring Overview](Powerbi-admin_api.png)


## 🛠️ Prerequisites: Service Principal Setup

To automate this extraction securely without relying on individual user accounts, you must configure a Service Principal according to Microsoft’s Admin API documentation:

1. **Microsoft Entra ID App**: Register an application in Azure to generate a Service Principal (Client ID and Secret).
2. **No Admin-Consent Permissions**: When running under Service Principal authentication, the app **must not** have any admin-consent required delegated permissions for Power BI set on it in the Azure portal.
3. **Power BI Admin Portal Configurations**: 
   A Fabric/Power BI Administrator must enable the following in the Tenant Settings:
   - **Developer Settings**: Enable *“Allow service principals to use Power BI APIs.”*
   - **Admin API Settings**: Enable *“Service principals can access read-only admin APIs.”*
   - **Metadata Scanning**: To extract deep schemas and DAX expressions, enable *“Enhance admin APIs responses with detailed metadata”* and *“Enhance admin APIs responses with DAX and mashup expressions.”*


## 🔍 Extracting Deep Metadata by Workspace ID

To look over granular, lower-level details, the most powerful tool is the **WorkspaceInfo Scanner API** (`Admin - WorkspaceInfo PostWorkspaceInfo`). This API initiates a scan for a requested list of workspace IDs.

**API Endpoint:**
```http
POST https://api.powerbi.com/v1.0/myorg/admin/workspaces/getInfo
```

### Passing Parameters for Lower-Level Details
By appending specific URI query parameters to this API request, you can force the extraction of highly granular metadata:

* `datasourceDetails=true`: Returns exact data source connection information (e.g., SQL server names, Web URLs).
* `datasetSchema=true`: Extracts table structures, column names, and measure names.
* `datasetExpressions=true`: Extracts the actual DAX and Mashup (Power Query) expressions used in the dataset.
* `getArtifactUsers=true`: Returns a list of users, Microsoft Entra groups, and Service Principals alongside their workspace roles (Admin, Member, Contributor, Viewer) for every report, dashboard, and dataset.
* `lineage=true`: Returns lineage information, including upstream dataflows and data source IDs.

*(Note: As per Microsoft Docs, after triggering the `PostWorkspaceInfo` request, you must call `GetScanStatus` to check completion, followed by `GetScanResult` to retrieve the final JSON payload).*

---

## 🔄 Monitoring Report Refreshes: Failures & Successes

To monitor the health of your datasets, the Admin APIs provide endpoints to extract refresh histories. 

**API Endpoints:**
```http
// Retrieve all refreshable items on a capacity
GET https://api.powerbi.com/v1.0/myorg/admin/capacities/refreshables

// Retrieve dataset-specific refresh history
GET https://api.powerbi.com/v1.0/myorg/datasets/{datasetId}/refreshes
```

**Data Points Extracted for the Report:**
* **Refresh Status:** Identifies if a refresh is `Completed`, `Failed`, `Disabled`, or `Unknown`.
* **Timestamps:** `startTime` and `endTime` in UTC format.
* **Performance Metrics:** Evaluates `averageDuration` to identify slow-running data models hogging capacity resources.
* **Failure Details:** The `serviceExceptionJson` property returns the exact JSON error code (e.g., outdated gateway, unreachable data source, or timeout errors) allowing you to troubleshoot directly from your monitoring report. Microsoft retains up to 60 recent refresh records per dataset.

---

## 📊 Building the Transparency & Governance Report

By orchestrating these API calls (e.g., using Azure Data Factory, Logic Apps, or Python) and storing the JSON outputs in a SQL Database or Data Lake, you can build a Power BI Dashboard over your own Power BI tenant. 

**Recommended Report Pages:**
1. **Governance & Security Matrix**: Use the `getArtifactUsers` data to build a matrix showing exactly who has access to which workspaces and reports. Easily audit for over-privileged users or external guests.
2. **Refresh Failure Triage**: A dashboard filtered by `status = Failed`. Extract the `serviceExceptionJson` to create an actionable punch-list for developers to fix broken data source credentials or gateway issues.
3. **Performance Monitoring**: A scatter chart of dataset `averageDuration` against the volume of data. Flag datasets that frequently hit capacity timeout limits.
4. **Data Dictionary / Lineage**: Use the `datasetSchema` and `lineage` parameters to allow users to search for specific DAX logic or column names across the entire tenant.

---

## 🧰 All Available Admin API Functionalities

As per the official Microsoft REST API Documentation, the Admin API surface supports comprehensive administration functionalities:

* **Workspace (Group) Administration**: Retrieve a list of all workspaces, add users as an admin, restore deleted workspaces, and run deep metadata scans (`PostWorkspaceInfo`).
* **Dataset Management**: Retrieve all datasets, view dataset query scale-out settings (`autoSyncReadOnlyReplicas`), get dataset users, update routing configurations, and pull data sources.
* **Refresh Management**: Retrieve all refreshable items on a capacity, check schedules, and pull refresh history logs.
* **Artifact Inventories**: Bulk extract lists of Reports, Dashboards, Dataflows, Datamarts, and Apps. 
* **Unused Artifacts**: Identify reports, dashboards, and datasets that have not been used within 30 days (`GetUnusedArtifactsAsAdmin`), helping to clean up tenant clutter.
* **Subscriptions Monitoring**: Retrieve subscription information (`GetUserSubscriptionsAsAdmin` and `GetReportSubscriptionsAsAdmin`) to see who is receiving automated emails.
* **Widely Shared Artifacts**: Use the `PublishedToWeb` endpoint to track items published to the public internet, ensuring no sensitive data is leaked.
* **Deployment Pipelines**: Get deployment pipelines and remove user permissions across environments.
* **Capacities Admin**: Modify specific capacity information and monitor capacity workloads.

---
> **Disclaimer:** All technical capabilities, limitations (e.g., maximum requests per hour), and parameters mentioned in this repository are sourced strictly from official **[Microsoft Learn Power BI REST API documentation](https://learn.microsoft.com/en-us/rest/api/power-bi/admin)**.
```
