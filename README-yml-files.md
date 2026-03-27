# ArkCase YAML Workflow (`chart-defaults` + `values`)

This repo uses two folders as the YAML source of truth:

- `C:\Users\benja_lfttkfq\Downloads\Github\arkcase-compose\chart-defaults`
- `C:\Users\benja_lfttkfq\Downloads\Github\arkcase-compose\values`

Use this model:

- `chart-defaults/` = regenerated, reference-only defaults from Helm charts
- `values/` = your custom runtime overrides used when deploying/redeploying ArkCase

---

## Why this structure matters

Keeping defaults and overrides separate gives you:

1. **Safe upgrades**: regenerate defaults any time without losing your custom config.
2. **Clean diffs**: you can quickly see what changed by chart version vs what your team changed.
3. **Repeatable deploys**: the app can always be rerun from the files in `values/`.
4. **Team consistency**: everyone uses the same local convention and file locations.

In short: `chart-defaults` is your baseline library; `values` is what actually runs.

---

## 1) Re-get all Helm defaults into `chart-defaults`

From repo root (`C:\Users\benja_lfttkfq\Downloads\Github\arkcase-compose`), refresh repo metadata first:

```powershell
helm repo add arkcase https://arkcase.github.io/ark_helm_charts
helm repo update
```

Then regenerate defaults for every `arkcase/*` chart:

```powershell
$ErrorActionPreference='Stop'
New-Item -ItemType Directory -Force -Path .\chart-defaults | Out-Null

$charts = helm search repo arkcase -o json |
  ConvertFrom-Json |
  Where-Object { $_.name -like 'arkcase/*' } |
  Select-Object -ExpandProperty name

foreach ($chart in $charts) {
  $short = $chart.Split('/')[1]
  $outFile = Join-Path .\chart-defaults ("{0}-values.yaml" -f $short)
  helm show values $chart | Out-File -FilePath $outFile -Encoding utf8
}

"Exported $($charts.Count) files to .\chart-defaults"
```

> Important: `helm show all arkcase` fails because `arkcase` is a **repo name**, not a full chart reference. Use `arkcase/app`, `arkcase/core`, etc.

---

## 2) Keep runtime overrides in `values`

Current runtime override file:

- `values/dev-values.yaml`

If you want more granular overrides, add more files under `values/` (examples):

- `values/00-global.yaml`
- `values/10-ingress.yaml`
- `values/20-core.yaml`
- `values/90-resources.yaml`

Do **not** edit files in `chart-defaults/` for runtime behavior; those are regenerated references.

---

## 3) Re-run ArkCase with the values you want

### Single-file deploy (current setup)

```powershell
helm upgrade --install arkcase arkcase/app -n arkcase --create-namespace -f .\values\dev-values.yaml
```

### Multi-file deploy (layered overrides)

```powershell
helm upgrade --install arkcase arkcase/app -n arkcase --create-namespace \
  -f .\values\00-global.yaml \
  -f .\values\10-ingress.yaml \
  -f .\values\20-core.yaml \
  -f .\values\90-resources.yaml
```

Helm merge behavior: if the same key appears in multiple files, the **last `-f` file wins**.

---

## 4) Validate before deploy

Render templates first to catch YAML or value issues:

```powershell
helm template arkcase arkcase/app -f .\values\dev-values.yaml | Out-File -FilePath .\rendered.yaml -Encoding utf8
```

For layered files, pass the same ordered `-f` list you plan to deploy with.

---

## 5) Quick operational rules

- Regenerate `chart-defaults/` whenever chart versions change.
- Keep all custom environment intent in `values/`.
- Prefer small, focused files in `values/` for large app configs.
- Commit both folders so your baseline + runtime intent are versioned together.

That’s the clean pattern: **re-get defaults into `chart-defaults`, run the app from `values`.** ✅
