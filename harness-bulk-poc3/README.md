# Harness CD bulk deployment POC — catalog-driven revision

This revision keeps all five services outside the pipeline YAML. The pipeline reads `services-poc.csv`, derives each GAR package from the standard naming convention, and repeats one standard Kubernetes deployment stage.

## What is defined once

| Setting | Value |
|---|---|
| Organization | `cse2` |
| Project | `obl11` |
| Environment | `gcp_rtl_pre` |
| Infrastructure | `ns_53e680_pre_fix_europe_west2_caas` |
| Target namespace | `ns-53e680-pre-fix` through the Infrastructure Definition |
| Artifact source identifier | `RTL_SSV_DOCKER_RELEASE_GAR` |
| GAR package prefix | `cse2/obl11/al08218/images` |
| GAR package naming rule | common prefix + `/ob-` + `deployment_name` |
| Artifact tag | `latest` |
| Base path | `/cse2/obl11` |
| Deploy environment | `pre_pre` |
| Git branch | `main` |
| Maximum concurrency | `2` |
| POC safety limit | `5` enabled services |

The GCP/GAR connector, GAR project, region, repository, Helm connector, chart and values files remain in each existing Harness Service definition. They are not repeated in the bulk pipeline.

## CSV format

Only three columns are required:

```csv
service_ref,deployment_name,enabled
svc_example_service,example-service,true
```

Set `enabled=false` to exclude a row. For the POC, the loader rejects more than five enabled rows.

## How package resolution works

For this row:

```csv
svc_aisp_balances_channel_service,aisp-balances-channel-service,true
```

the pipeline derives:

```text
cse2/obl11/al08218/images/ob-aisp-balances-channel-service
```

Harness then uses the existing Google Artifact Registry Primary Artifact to fetch the `latest` tag from that package.

## One-time setup

1. Commit `services-poc.csv` to a Git repository reachable from the Harness Delegate.
2. Copy its raw-file URL.
3. Replace `REPLACE_ONCE_WITH_RAW_CSV_URL` in the pipeline YAML with that raw URL.
4. If the Git raw URL requires authentication, configure access on the Delegate or adapt the `curl` line to reference an approved Harness secret. Do not put a token in the CSV or YAML.
5. Import `harness-bulk-deploy-5-services.yaml` into Harness project `obl11`.

## Required checks before the first run

1. Confirm all five Harness Services exist with the exact identifiers in the CSV.
2. Confirm the Primary Artifact identifier is exactly `RTL_SSV_DOCKER_RELEASE_GAR` for all five Services.
3. Confirm every package follows the `/ob-<deployment_name>` rule.
4. Confirm every selected GAR package has the literal `latest` tag. The prior execution screenshot showed a generated Version value, so this cannot be assumed.
5. Keep preflight checks enabled.

## Pipeline execution

1. `Load Service Catalog` downloads and validates the CSV.
2. It selects only rows where `enabled=true`.
3. It derives a compact list containing service reference, GAR package and common version.
4. `Deploy Service` repeats once for each list item.
5. Harness deploys a maximum of two service iterations concurrently.
6. Every failed iteration uses Stage Rollback.

## Scaling after the POC

After the five-service run succeeds, increase the safety limit gradually, raise concurrency only after capacity testing, and replace literal `latest` with immutable digest resolution before production use.

## Harness compatibility note

The pipeline uses a supported Harness pattern: a Shell Script output variable is converted back into a list with `.split(",")` and supplied to a repeat strategy. Final validation must occur in your Harness account because the exact Service runtime-input schema is generated from your existing Service definitions.
