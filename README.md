# Financial summary dashboard

The instructions below describe how to set up a dashboard that provides a financial summary for your business units. The dashboard is designed to help monitor costs and usage patterns effectively. It has four sections:
1. ECH billing summary
2. ECH credit burndown (not filtered by deployment group)
3. Deployment usage summary
4. Chargeback summary

**Known Issue:** Usage data filtering by deployment group is not yet functional

## Dashboard Versions

- **v3** (recommended): Uses `ess.billing.deployment_tags` with `chargeback_group` key. No custom pipelines required. 
- **v2/v1**: Legacy versions using deployment name parsing (requires custom pipelines)

## Dependencies

1. [Elasticsearch Service Billing integration](https://www.elastic.co/docs/reference/integrations/ess_billing) with `ess.billing.deployment_tags` field support
2. [Elasticsearch integration](https://www.elastic.co/docs/reference/integrations/elasticsearch) usage transform
3. [Elasticsearch Chargeback integration](https://github.com/elastic/elasticsearch-chargeback/tree/main/integration)
4. Deployments must have a tag with key `chargeback_group` to specify the deployment group

## Steps to implement the dashboard (v3)

1. Ensure your deployments are tagged with `chargeback_group` key in ECH
2. Verify the ESS Billing integration is collecting `ess.billing.deployment_tags` field
3. Import the [v3 dashboard NDJSON](https://github.com/JohannesMahne/fin_overview_dashboard/blob/main/fin-overview-dashboard-v3.ndjson)
4. The dashboard will automatically use the `chargeback_group` tag values to filter billing and chargeback data

## Migrating from v1/v2 to v3

If you're upgrading from the legacy dashboard versions, you can clean up the custom pipelines and enrichment policy that are no longer needed:

```
# Remove custom ingest pipelines
DELETE _ingest/pipeline/metrics-ess_billing.billing@custom
DELETE _ingest/pipeline/metrics-elasticsearch.ingest_pipeline@custom

# Remove enrichment policy
DELETE /_enrich/policy/fin_dashboard_deployment_lookup

# Remove watcher (if configured)
DELETE _watcher/watch/execute_fin_dashboard_deployment_lookup_enrich_policy

# Remove enrich user and role (if configured)
DELETE /_security/user/enrich-watcher-user
DELETE /_security/role/enrichment_policy_role
```

After cleanup, follow the v3 setup steps above.

---

## Legacy Setup (v1/v2 - using custom pipelines)

The following instructions are for the legacy dashboard versions that require custom ingest pipelines to extract deployment group from deployment names.

### Steps to implement legacy dashboard

1. Set up an enrichment policy to map deployment IDs to deployment names for usage data
2. Build a custom ingest pipeline to extract the business unit from the deployment name and set it as `deployment_group`
3. Build a custom ingest pipeline for usage data, using the enrichment policy to add deployment names and extract the business unit as `deployment_group`
4. Update historical billing and usage data by increasing the lookback value or running an update-by-query operation to apply the new pipelines
5. Upgrade the Chargeback integration to ensure compatibility with the new pipelines
6. Set up a watcher to execute the enrichment policy daily and keep the enrich index up to date
7. Import the legacy [dashboard NDJSON](https://github.com/JohannesMahne/fin_overview_dashboard/blob/main/fin-overview-dashboard.ndjson)

---

## Legacy Pipeline Details

### Create enrichment policy

This policy will enrich the `elasticsearch.cluster.name` (which is just the deployment ID / cluster ID) field with the `deployment_group`. It will be used on the usage data to apply the same deployment group as the billing data.

```
# Create the enrich index from the billing_cluster_cost_lookup index
PUT /_enrich/policy/fin_dashboard_deployment_lookup
{
  "match": {
    "indices": "billing_cluster_cost_lookup",
    "match_field": "deployment_id",
    "enrich_fields": ["deployment_group"]
  }
}

# Execute the enrichment policy to populate the enrich index
POST /_enrich/policy/fin_dashboard_deployment_lookup/_execute
```

### Create ingest pipeline for billing data

The ingest pipeline will extract the `business_unit` from the `deployment_name` and set it as `deployment_group`. This will be used in the dashboard to filter by business unit.

If the `ess.billing.deployment_tags` field contains a tag with key `chargeback_group`, its value will be used as `deployment_group`. Otherwise, the pipeline falls back to extracting the business unit from the deployment name.

Using the `@custom` suffix to avoid overwriting the default pipeline provided by Elastic.

```
PUT _ingest/pipeline/metrics-ess_billing.billing@custom
{
  "description": "Set deployment group from chargeback_group tag",
  "processors": [
    {
      "script": {
        "description": "Extract chargeback_group from deployment_tags if available",
        "if": "ctx?.ess?.billing?.deployment_tags != null && ctx.ess.billing.deployment_tags instanceof List",
        "lang": "painless",
        "source": """
          for (def tag : ctx.ess.billing.deployment_tags) {
            if (tag.key == 'chargeback_group') {
              ctx.deployment_group = tag.value;
              break;
            }
          }
        """
      }
    },
    {
      "set": {
        "field": "deployment_group",
        "if": "ctx?.deployment_group == null",
        "value": "unspecified"
      }
    }
  ]
}
```

To update historical data, go to the integration and change the lookback value to a higher value (e.g. 366 days) and save the integration. This will reprocess the historical data using the new ingest pipeline.

### Create ingest pipeline for usage data

The usage data does not have the `deployment_group` field, so we will use the enrich processor to get the `deployment_group` from the `elasticsearch.cluster.name` field using the enrichment policy.

Using the `@custom` suffix to avoid overwriting the default pipeline provided by Elastic.

```
# Create the ingest pipeline
PUT _ingest/pipeline/metrics-elasticsearch.ingest_pipeline@custom
{
  "description": "Enrich usage data with deployment group from billing",
  "processors": [
    {
      "enrich": {
        "if": "ctx?.elasticsearch?.cluster?.name != null",
        "policy_name": "fin_dashboard_deployment_lookup",
        "field": "elasticsearch.cluster.name",
        "target_field": "enriched",
        "max_matches": 1
      }
    },
    {
      "set": {
        "if": "ctx?.enriched?.deployment_group != null",
        "field": "deployment_group",
        "value": "{{enriched.deployment_group}}"
      }
    },
    {
      "remove": {
        "field": "enriched",
        "ignore_missing": true
      }
    },
    {
      "set": {
        "field": "deployment_group",
        "if": "ctx?.deployment_group == null",
        "value": "unspecified"
      }
    }
  ]
}
```

To update historical data, run the following query to reprocess all documents in the `monitoring-indices` index using the new ingest pipeline. This can be done on `monitoring-indices` because it is not a data stream, but a regular index.

```
# Update historical data with the new ingest pipeline
POST monitoring-indices/_update_by_query?pipeline=metrics-elasticsearch.ingest_pipeline@custom&wait_for_completion=false
{
  "query": {
    "match_all": {}
  }
}
```

### Create watcher to execute the enrich policy daily

To keep the enrich index up to date with any new deployments, we will create a watcher that executes the enrich policy daily at 2:15am. This requires creating a role and user with the necessary permissions to execute the enrich policy.

```
# Create role and user for watcher to execute the enrich policy
PUT /_security/role/enrichment_policy_role
{
  "cluster": ["manage_enrich"],
  "indices": [
    {
      "names": ["billing_cluster_cost_lookup"],
      "privileges": ["view_index_metadata", "read"]
    }
  ]
}

# Create user with the above role
# Make sure to replace {SELECT_YOUR_OWN_PASSWORD} with a strong password of your choice
POST /_security/user/enrich-watcher-user
{
  "password": "{SELECT_YOUR_OWN_PASSWORD}",
  "roles": ["enrichment_policy_role"],
  "full_name": "Enrich Watcher User",
  "email": "enrich-watcher-user@example.com"
}

# Create watcher to execute the enrich policy daily at 2:15am
# Make sure to replace {ELASTIC_ENDPOINT} with your actual Elastic endpoint URL
# and {SELECT_YOUR_OWN_PASSWORD} with the password you set for enrich-watcher
PUT _watcher/watch/execute_fin_dashboard_deployment_lookup_enrich_policy
{
    "trigger" : {
        "schedule" : { "daily" : { "at" : "02:15" } } 
    },
    "input" : {
        "http" : {
            "request" : {
                "url" : "{ELASTIC_ENDPOINT}/_enrich/policy/fin_dashboard_deployment_lookup/_execute",
                "method": "post",
                "auth" : {
                    "basic" : {
                    "username" : "enrich-watcher-user",
                    "password" : "{SELECT_YOUR_OWN_PASSWORD}"
                    }
                }
            }
        }
    }
}
```

### Upload dashboard

Upload the [dashboard NDJSON](https://github.com/JohannesMahne/fin_overview_dashboard/blob/main/fin-overview-dashboard.ndjson) file.

### Changelog

- v1: Initial release with deployment name parsing
- v2: Simplified pipelines, uses tags only
- v3: ES|QL controls with `?deployment_group` parameter, tag-based filtering
- v4: Updated to `conf_chargeable_unit_rate` field, null-check on all billing queries for empty control state
- v5: Monitoring visualizations now filter by deployment_group via LOOKUP JOIN