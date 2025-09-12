# Financial summary dashboard

The instructions below describe how to set up a dashboard that provides a financial summary for your business units. The dashboard is designed to help monitor costs and usage patterns effectively. It has four sections:
1. ECH billing summary
2. ECH credit burndown (not filtered by deployment group)
3. Deployment usage summary
4. Chargeback summary

## Dependencies

1. [Elasticsearch Service Billing integration](https://www.elastic.co/docs/reference/integrations/ess_billing)
2. [Elasticsearch integration](https://www.elastic.co/docs/reference/integrations/elasticsearch) usage transform
3. [Elasticsearch Chargeback integration](https://github.com/elastic/elasticsearch-chargeback/tree/main/integration)
4. Deployment names in ECH should be in the format: `<company_name>_<country_code>_<business_unit>_<environment>_<function>`

## Steps to implement the dashboard

1. Set up an enrichment policy to map deployment IDs to deployment names for usage data.
2. Build a custom ingest pipeline to extract the business unit from the deployment name and set it as `deployment_group`.
3. Build a custom ingest pipeline for usage data, using the enrichment policy to add deployment names and extract the business unit as `deployment_group`.
4. Update historical billing and usage data by increasing the lookback value or running an update-by-query operation to apply the new pipelines.
5. Upgrade the Chargeback integration to ensure compatibility with the new pipelines.
6. Set up a watcher to execute the enrichment policy daily and keep the enrich index up to date.
7. Import the [dashboard NDJSON](https://github.com/JohannesMahne/fin_overview_dashboard/blob/main/fin-overview-dashboard.ndjson) to use the `deployment_group` field to filter by business

## Details

### Create enrichment policy

This policy will enrich be used to enrich the `elasticsearch.cluster.name` (which is just the deployment ID / cluster ID) field with the `deployment_name`. It will be used on the usage data that does not have the deployment name.

```
# Create the enrich index from the billing_cluster_cost_lookup index
PUT /_enrich/policy/fin_dashboard_deployment_lookup
{
  "match": {
    "indices": "billing_cluster_cost_lookup",
    "match_field": "deployment_id",
    "enrich_fields": ["deployment_name"]
  }
}

# Execute the enrichment policy to populate the enrich index
POST /_enrich/policy/fin_dashboard_deployment_lookup/_execute
```

### Create ingest pipeline for billing data

The ingest pipeline will extract the `business_unit` from the `deployment_name` and set it as `deployment_group`. This will be used in the dashboard to filter by business unit.

Using the `@custom` suffix to avoid overwriting the default pipeline provided by Elastic.

```
PUT _ingest/pipeline/metrics-ess_billing.billing@custom
{
  "description": "Unify deployment group as business_unit for billing and monitoring",
  "processors": [
    {
      "set": {
        "field": "deployment_name",
        "if": "ctx?.ess?.billing?.deployment_name != null",
        "value": "{{ess.billing.deployment_name}}"
      }
    },
    {
      "grok": {
        "field": "deployment_name",
        "patterns": [
          "(?<company_name>[^_]+)_(?<country_code>[^_]+)_(?<business_unit>[^_]+)_(?<env>[^_]+)(?:_(?<solution>.*))?"
        ],
        "ignore_failure": true
      }
    },
    {
      "set": {
        "field": "deployment_group",
        "if": "ctx?.business_unit != null",
        "value": "{{business_unit}}"
      }
    }
  ]
}
```

To update historical data, go to the integration and change the lookback value to a higher value (e.g. 366 days) and save the integration. This will reprocess the historical data using the new ingest pipeline.

### Create ingest pipeline for usage data

The usage data does not have the `deployment_name` field, so we will use the enrich processor to get the `deployment_name` from the `elasticsearch.cluster.name` field. Then we will extract the `business_unit` from the `deployment_name` and set it as `deployment_group`.

Using the `@custom` suffix to avoid overwriting the default pipeline provided by Elastic.

```
# Create the ingest pipeline
PUT _ingest/pipeline/metrics-elasticsearch.ingest_pipeline@custom
{
  "description": "Unify deployment group as business_unit for billing and monitoring",
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
        "if": "ctx?.enriched?.deployment_name != null",
        "field": "deployment_name",
        "value": "{{enriched.deployment_name}}"
      }
    },
    {
      "remove": {
        "field": "enriched",
        "ignore_missing": true
      }
    },
    {
      "grok": {
        "field": "deployment_name",
        "patterns": [
          "(?<company_name>[^_]+)_(?<country_code>[^_]+)_(?<business_unit>[^_]+)_(?<env>[^_]+)(?:_(?<solution>.*))?"
        ],
        "ignore_failure": true
      }
    },
    {
      "set": {
        "if": "ctx?.business_unit != null",
        "field": "deployment_group",
        "value": "{{business_unit}}"
      }
    }
  ]
}
```

To update historical data, run the following query to reprocess all documents in the `monitoring-indices` index using the new ingest pipeline. This can be done on `monitoring-indices` because it is not a data stream, but a regular index.

```
# Update historical data with the new ingest pipeline
POST monitoring-indices/_update_by_query?pipeline=metrics-elasticsearch.ingest_pipeline@custom
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
