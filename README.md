# License Optimizer — Deployment

Microsoft 365 licensing and cost governance solution: Dataverse, Power Automate, Azure Functions, Power BI, and Copilot Studio.

## Downloads

| File | What it is | Download |
|---|---|---|
| Azure Functions package | Function code (used automatically by the Deploy button) | [LicenseOptimizerFunctions.zip](https://github.com/gitmahesh91/license-optimizer-deploy/releases/download/v1.0.0/LicenseOptimizerFunctions.zip) |
| Power Platform solution | Tables, flows, Copilot agents | [LicenseOptimizerUnmanaged_1_0_0_2.zip](https://github.com/gitmahesh91/license-optimizer-deploy/releases/download/solution-v1.0.0/LicenseOptimizerUnmanaged_1_0_0_2.zip) |
| Power BI report | License dashboards | [PowerBI_License_Optimizer.pbix](https://github.com/gitmahesh91/license-optimizer-deploy/releases/download/solution-v1.0.0/PowerBI_License_Optimizer.pbix) |

## Before you start

Have these ready:

| Item | Where to find it |
|---|---|
| Tenant ID | Entra ID → Overview → Tenant ID |
| Client ID | Entra ID → App registrations → your app → Application (client) ID |
| Client Secret | Your app registration → Certificates & secrets (the secret **value**) |
| Dataverse URL | Power Platform admin center → your environment → Environment URL |
| SharePoint site URL | The site where the License Master list will live |

Also make sure:
- The app registration has the required Microsoft Graph permissions with **admin consent** granted.
- The app registration is added as an **Application User** (with a security role) in the target Dataverse environment.
- The **TDS endpoint** is enabled on the environment (Power Platform admin center → environment → Settings → Product → Features). Power BI needs it.
- **Power BI Desktop** is installed.

---

## Step 1 — Deploy the Azure Functions

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fgitmahesh91%2Flicense-optimizer-deploy%2Fmain%2Fazuredeploy.json)

1. Click **Deploy to Azure** and sign in.
2. **Subscription** and **Resource group**: select yours.
3. **Location**: type your region, e.g. `centralindia`.
4. **Tenant Id / Client Id / Client Secret / Dataverse Url**: paste your values.
5. **Existing Plan Name**:
   - Leave empty to create a new Consumption plan.
   - If you get a *quota* error (`SubscriptionIsOverQuotaForSku`), enter the exact name of an existing Consumption (Y1) plan in the same resource group, and set **Location** to that plan's region.
6. Leave the other fields as default → **Review + create** → **Create**.
7. When complete, open **Outputs** → copy **functionAppUrl**.
8. Open the new Function App → **Functions** → confirm all **12 functions** are listed.

What gets created: Function App (.NET 8 isolated, Functions v4, Windows), Consumption plan (unless reused), Storage account, Application Insights + Log Analytics.

## Step 2 — Import the Power Platform solution

1. Download **LicenseOptimizerUnmanaged_1_0_0_2.zip** (link above).
2. Go to **make.powerapps.com** → select your environment → **Solutions** → **Import solution**.
3. **Browse** → select the zip → **Next**.
4. On the **Connections** screen, select or create a connection for each connection reference → **Import**.
5. Wait for the import to finish, then open the solution and confirm the tables, flows, and agents are there.

## Step 3 — Fill the Config record

1. In the solution, open the **Config** table → **Edit data** (or add a row).
2. Create **one** row with:

| Column | Value |
|---|---|
| Config Key | `MAIN` |
| SharePoint Site Url | your SharePoint site URL |
| Dataverse Url | your environment URL |
| Function App Url | the **functionAppUrl** from Step 1 |

3. Save.

## Step 4 — Run the Setup Flow

1. In the solution, open **LO-Setup-Orchestrator** → **Run**.
2. It runs these steps in order: precondition check → read config → create SharePoint list → seed reference data → set environment variables → Power BI refresh → validate.
3. When it finishes, open the Config row → read **Last Setup Result**:
   - **SUCCESS** → continue.
   - **FAILED at …** → it names the step and the reason. Fix it and run the flow again. Re-running is safe.

> Power BI is skipped in the first run. That's expected — it runs after Step 5.

## Step 5 — Set up Power BI

1. Download **PowerBI_License_Optimizer.pbix** and open it in **Power BI Desktop**.
2. **Home → Transform data → Data source settings**.
3. Select the Dataverse source → **Change Source** → enter your environment URL **without** `https://` (e.g. `yourorg.crm8.dynamics.com`) → **OK** → **Close**.
4. **Close & Apply** → sign in when asked → wait for the data to load.
5. **Publish** → select your workspace.
6. In the Power BI service → workspace → the semantic model → **Settings** → **Data source credentials** → **Edit credentials** → sign in (OAuth2).
7. Copy the **workspace ID** and **dataset (semantic model) ID** from the browser URL:
   `app.powerbi.com/groups/<workspaceId>/datasets/<datasetId>/...`
8. Paste them into the Config row (**Power BI Workspace Id**, **Power BI Dataset Id**).
9. Run **LO-Setup-Orchestrator** again — it now refreshes the dataset.

## Step 6 — Go live

1. Turn on the operational flows in the solution.
2. Open **Copilot Studio** → open each License Optimizer agent → **Publish**.
3. Confirm the first data sync runs (Function App → Functions → **Invocations**).

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Azure: `SubscriptionIsOverQuotaForSku` | Use **Existing Plan Name** (Step 1.5) |
| Azure: `Cannot find serverFarm with name …` | Plan name typo or plan in another resource group. Copy the name exactly from the plan's page. |
| Function App shows 0 functions | Open the Function App → **Functions** → **Refresh**. If still empty, **Restart** the app. |
| Child flow fails with a "run-only user" connection error | Open that child flow → **Run only users** → set each connection to **Use this connection** |
| Setup fails at "Read Config" | A required Config value is empty. Last Setup Result lists which one. |
| Power BI refresh fails | Set **Data source credentials** in the Power BI service (Step 5.6) |
