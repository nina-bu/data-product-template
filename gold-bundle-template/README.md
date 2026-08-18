# gold-bundle-template

A Databricks Asset Bundle **template** for gold-layer data products with Azure DevOps
CI/CD baked in. The platform team owns this repo; business users scaffold new
products from it — they never hand-write bundle or pipeline config.

## What a scaffolded product gets
- `databricks.yml` with `dev` / `staging` / `prod` targets (dev = development mode,
  staging & prod = production mode), catalog/schema/warehouse filled from your answers.
- `azure-pipelines.yml` — the promotion pipeline: **PR runs unit tests + validates →
  merge deploys to staging + runs an integration test → manual approval gate → prod**.
- A starter job that builds a gold table, plus `src/` to edit — pure logic in
  `src/transformations.py`, a thin notebook in `src/build_gold.py`.
- A `tests/` folder of pytest unit tests for the transform logic, run in the
  pipeline's PR/validate stage (no cluster needed).

## Create a new data product (business user)

### Before you start: authentication
`bundle init` renders files locally and never calls a workspace — but the Databricks
CLI still requires a valid auth context to exist before it will run any command. So:

- **Be authenticated to a workspace you can reach** (e.g. `databricks configure` or
  `databricks auth login`, or `export DATABRICKS_CONFIG_PROFILE=<profile>`). If your
  default profile points at a dead host, `init` fails with a token/OAuth error — pass a
  working profile instead.
- **Authenticate to the dev workspace you'll actually deploy to.** The profile you
  authenticate with is *independent* of the `workspace_host` you type at the prompt —
  but using the real dev workspace means `databricks bundle validate/deploy -t dev`
  works right after scaffolding.
- **Enter the real workspace URL** at the `workspace_host` prompt
  (`https://adb-….azuredatabricks.net`). Whatever you type is written verbatim into
  `databricks.yml` for all three targets.

Scaffold directly inside a fresh, empty repo so the files land at the repo root:

```bash
# 1. Create an EMPTY repo in Azure DevOps (no README/.gitignore) — call it after
#    the data product, e.g. finance_gold.

# 2. Clone it and go in.
git clone https://dev.azure.com/<org>/<project>/_git/finance_gold
cd finance_gold

# 3. Scaffold from this template. You'll be prompted for product_name,
#    workspace_host, catalog, schema, warehouse_id.
databricks bundle init https://dev.azure.com/<org>/<project>/_git/gold-bundle-template

# 4. Commit and push what was generated.
git add . && git commit -m "Scaffold from gold-bundle-template" && git push
```

Notes:
- Start from an **empty** repo — the template writes a `README.md`/`.gitignore` at the
  root and won't overwrite existing files.
- Pin a template version with `--tag v1.2.0` (or `--branch`) so evolving the template
  doesn't change already-scaffolded repos.
- Then develop locally: `databricks bundle validate -t dev`,
  `databricks bundle deploy -t dev`, `databricks bundle run <product>_build -t dev`.

## Per-repo wiring (platform team, once per new repo)

`bundle init` produces repo **content**. The following are Azure DevOps *project*
resources, so a project admin sets them once for each new product repo. The service
principal, its Unity Catalog grants, and the `databricks-sp` variable group are
configured once at the project level and reused by every scaffolded repo.

### 1. Register the pipeline
The `azure-pipelines.yml` in the repo is just a file — a recipe. Azure DevOps won't
run it until you register a pipeline object that points at it:

> **Pipelines → New pipeline → Azure Repos Git → pick the repo →
> "Existing Azure Pipelines YAML file" → select `/azure-pipelines.yml` → Save.**

Registration is what creates the runnable pipeline (its run history, permissions, and
access to the variable group and environments). After this, the `trigger:` and `pr:`
rules in the file fire automatically on PRs and merges to `main`.

### 2. Create the environments and the prod approval gate
The pipeline's staging and prod stages target `environment: staging` and
`environment: prod`. An *Environment* is a named object you attach rules to:

> **Pipelines → Environments → New environment → name it `staging`.** Repeat for `prod`.
> (Names must match the YAML exactly.)

Then add the approval check on prod:

> **Environments → `prod` → Approvals and checks → Approvals → add the platform
> team as approver(s) → Save.**

Now every time the pipeline reaches the prod stage it **pauses and waits** until a
named approver clicks Approve before `databricks bundle deploy -t prod` runs. `staging`
needs no approval. Optionally restrict which branches/pipelines may deploy to prod.

### 3. Protect `main`
> **Repos → Branches → `main` → Branch policies:** require a pull request, and require
> the `validate` build to pass before merge. This blocks direct pushes and enforces the
> PR gate.

## Maintaining the template
- Edit files under `template/` (they are Go templates; `{{`.`}}` markers are
  substituted at init time, and the `.tmpl` suffix is stripped from output names).
- Edit `databricks_template_schema.json` to change the questions asked.
- After any change, scaffold a throwaway product and run `databricks bundle validate`
  to confirm the template still produces a valid bundle:

  ```bash
  databricks bundle init . --config-file answers.json --output-dir /tmp/try
  ```
