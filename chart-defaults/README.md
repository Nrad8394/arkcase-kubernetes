# Chart Defaults Reference (`chart-defaults/`)

This folder contains exported Helm default values for ArkCase charts.

## What this folder is for

- **Reference baseline** for each ArkCase chart.
- **Diff source** when chart versions change.
- **Input for planning overrides** in `../values/`.

## What this folder is NOT for

- Do **not** treat these files as runtime overrides.
- Do **not** hand-edit these files and expect stable long-term results.

Use `../values/` for active deployment configuration.

---

## How to regenerate all files in this folder

From repo root (`C:\Users\benja_lfttkfq\Downloads\Github\arkcase-compose`):

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
```

> Reminder: `helm show all arkcase` fails because `arkcase` is a repo name, not a full chart reference.

---

## File-by-file purpose

| File | Helm chart | What it does |
|---|---|---|
| `activemq-values.yaml` | `arkcase/activemq` | Defaults for ActiveMQ messaging used by ArkCase. |
| `alfresco-values.yaml` | `arkcase/alfresco` | Defaults for Alfresco Content Services deployment. |
| `app-values.yaml` | `arkcase/app` | Top-level ArkCase application chart defaults (umbrella-style entry point). |
| `artemis-values.yaml` | `arkcase/artemis` | Defaults for Apache Artemis messaging broker. |
| `cloudconfig-values.yaml` | `arkcase/cloudconfig` | Defaults for ArkCase cloud/config distribution settings. |
| `common-values.yaml` | `arkcase/common` | Shared chart library defaults/templates consumed by other ArkCase charts. |
| `content-values.yaml` | `arkcase/content` | Defaults for content storage services (e.g., Alfresco/MinIO pattern). |
| `content-migrator-values.yaml` | `arkcase/content-migrator` | Defaults for migration from Alfresco content to S3-style storage. |
| `core-values.yaml` | `arkcase/core` | Defaults for core ArkCase services and ConfigServer. |
| `database-values.yaml` | `arkcase/database` | Defaults for database engine abstraction (PostgreSQL or MariaDB options). |
| `gateway-apache-values.yaml` | `arkcase/gateway-apache` | Defaults for Apache reverse proxy gateway in front of ArkCase services. |
| `hostpath-provisioner-values.yaml` | `arkcase/hostpath-provisioner` | Defaults for local hostPath dynamic PV provisioning. |
| `mariadb-values.yaml` | `arkcase/mariadb` | Defaults for MariaDB database deployment. |
| `neo4j-values.yaml` | `arkcase/neo4j` | Defaults for Neo4j graph database component. |
| `pentaho-values.yaml` | `arkcase/pentaho` | Defaults for Pentaho reporting/BI component. |
| `postgres-values.yaml` | `arkcase/postgres` | Defaults for PostgreSQL deployment. |
| `pvc-tool-values.yaml` | `arkcase/pvc-tool` | Defaults for utility pod that mounts/copies PVC data (rsync/clone workflows). |
| `samba-values.yaml` | `arkcase/samba` | Defaults for Samba-based file sharing integration. |
| `snowbound-values.yaml` | `arkcase/snowbound` | Defaults for Snowbound document viewing integration. |
| `solr-values.yaml` | `arkcase/solr` | Defaults for Solr search/indexing component. |
| `step-ca-values.yaml` | `arkcase/step-ca` | Defaults for Step-CA certificate authority service. |
| `zookeeper-values.yaml` | `arkcase/zookeeper` | Defaults for ZooKeeper coordination service. |

---

## How this connects to `values/`

Recommended workflow:

1. Regenerate `chart-defaults/` when chart versions change.
2. Compare defaults to your runtime needs.
3. Put only your intended overrides in `../values/*.yaml`.
4. Deploy with `helm upgrade --install ... -f ../values/<file>.yaml` (or multiple `-f` files).

This keeps upgrades clean and overrides intentional.
