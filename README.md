# data-product-template

Custom Databricks Asset Bundle (DABs) template(s) for gold-layer data products,
with Azure DevOps CI/CD baked in.

Each template lives in its **own top-level subfolder** (with
`databricks_template_schema.json` at that subfolder's root). This layout is what
the Databricks **workspace UI** requires — it scans this folder and treats each
subfolder as a template.

## Templates

- [`gold-bundle-template/`](gold-bundle-template/) — gold-layer data product:
  `dev` / `staging` / `prod` targets, pure logic in `src/transformations.py` with
  pytest **unit tests**, an **integration test** in staging, and an Azure DevOps
  promotion pipeline (**PR → unit tests + validate → merge → staging + test →
  approval → prod**). See its [README](gold-bundle-template/README.md).

## Use it — from the Databricks workspace UI

1. Clone this repo as a **Git folder** in your workspace.
2. Admin: **Settings → Development → Declarative Automation Bundles** → select this
   repo's folder as the custom-templates folder.
3. Create a new project, pick **gold-bundle-template**, and answer the 5 prompts.

## Use it — from the CLI

The schema is in a subfolder, so point `--template-dir` at it:

```bash
databricks bundle init https://github.com/nina-bu/data-product-template \
  --template-dir gold-bundle-template
```

(You must be authenticated to a reachable workspace first — `bundle init` needs a
valid auth profile even though it renders locally.)
