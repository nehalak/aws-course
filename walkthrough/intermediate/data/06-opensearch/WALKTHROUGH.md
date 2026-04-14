# Walkthrough — 06 OpenSearch

## About this service
**OpenSearch** is AWS's fork of Elasticsearch + Kibana (now "OpenSearch Dashboards"), offered as a managed service. Three flavors: **managed domains** (provisioned clusters you size), **OpenSearch Serverless** (pay-per-OCU, auto-scales — ideal for vector search), and **self-managed on EC2**. Used for full-text search, log analytics, observability, and vector (k-NN) search for RAG.

**Why it matters:** when Postgres/DynamoDB queries don't cut it — you need relevance scoring, fuzzy search, aggregations across billions of log lines, or vector similarity for LLM retrieval.

**When to use:** log analytics (CloudWatch Logs → OpenSearch), product search, dashboards, k-NN / semantic search, SIEM.
**When NOT to use:** primary system of record (no transactions), tiny data (S3 + Athena is cheaper), simple keyword search (Postgres `tsvector` is fine).

## Estimated cost
- **`t3.small.search` (dev only): ~$25/month** per node
- **`r6g.large.search` (prod minimum): ~$96/month** per node + $0.122/GB/mo EBS
- **3-node prod cluster: ~$290/month** + storage
- **OpenSearch Serverless: $0.24/OCU-hour** (min 2 OCU indexing + 2 search = **~$700/month floor**)
- **UltraWarm nodes: ~60% cheaper** than hot for read-mostly
- Total for this lesson: **~$25/month** dev domain if destroyed same day.

---

## Step 1: OpenSearch domain in VPC with fine-grained auth
> **Why:** Start dev-sized: 1 node, VPC-only (no public internet), fine-grained access control with a master user stored in Secrets Manager. Production adds multi-AZ (3 dedicated masters + data nodes), encryption, UltraWarm.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as os from 'aws-cdk-lib/aws-opensearchservice';
import * as sm from 'aws-cdk-lib/aws-secretsmanager';

export class OpenSearchStack extends cdk.Stack {
  constructor(scope: any, id: string, props: cdk.StackProps & { vpc: ec2.IVpc }) {
    super(scope, id, props);
    const { vpc } = props;

    const masterCreds = new sm.Secret(this, 'MasterCreds', {
      generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: 'admin' }),
        generateStringKey: 'password',
        excludePunctuation: true,
        passwordLength: 24,
      },
    });

    const domain = new os.Domain(this, 'Search', {
      version: os.EngineVersion.OPENSEARCH_2_13,
      capacity: {
        dataNodes: 1,
        dataNodeInstanceType: 't3.small.search',
      },
      ebs: { volumeSize: 10, volumeType: ec2.EbsDeviceVolumeType.GP3 },
      vpc,
      vpcSubnets: [{ subnets: [vpc.privateSubnets[0]] }],
      zoneAwareness: { enabled: false },     // single-AZ for dev
      enforceHttps: true,
      nodeToNodeEncryption: true,
      encryptionAtRest: { enabled: true },
      fineGrainedAccessControl: {
        masterUserName: 'admin',
        masterUserPassword: masterCreds.secretValueFromJson('password'),
      },
      accessPolicies: [
        new cdk.aws_iam.PolicyStatement({
          actions: ['es:*'],
          principals: [new cdk.aws_iam.AnyPrincipal()],
          resources: ['*'],
        }),
      ],
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    new cdk.CfnOutput(this, 'DomainEp', { value: domain.domainEndpoint });
    new cdk.CfnOutput(this, 'SecretArn', { value: masterCreds.secretArn });
  }
}
```

Deploy takes ~15-20 min.

## Step 2: Bulk index from bastion
> **Why:** `_bulk` is the high-throughput ingest API — thousands of docs in one HTTP call. Index mapping defines field types; getting it right up-front prevents later reindexing pain.

```bash
# Create index with mapping
curl -u admin:$PASS -XPUT "https://$EP/products" -H 'Content-Type: application/json' -d '{
  "mappings": {
    "properties": {
      "name":  { "type": "text" },
      "price": { "type": "float" },
      "tags":  { "type": "keyword" },
      "createdAt": { "type": "date" }
    }
  }
}'

# Bulk load
cat <<EOF > bulk.ndjson
{"index":{"_index":"products","_id":"1"}}
{"name":"blue t-shirt","price":19.99,"tags":["apparel","sale"],"createdAt":"2026-03-01"}
{"index":{"_index":"products","_id":"2"}}
{"name":"red hoodie","price":49.99,"tags":["apparel"],"createdAt":"2026-03-05"}
EOF

curl -u admin:$PASS -XPOST "https://$EP/_bulk" \
  -H 'Content-Type: application/x-ndjson' --data-binary @bulk.ndjson
```

Expected: response `{"took":..., "errors": false, "items":[...]}`

## Step 3: Queries
> **Why:** `match` = analyzed full-text search (BM25 scored). `term` = exact keyword match (no analyzer). Aggregations compute facets, histograms, top-N without pulling docs client-side.

```bash
# Full-text
curl -u admin:$PASS -XGET "https://$EP/products/_search" -H 'Content-Type: application/json' -d '{
  "query": { "match": { "name": "hoodie" } }
}'

# Aggregation: price histogram + count per tag
curl -u admin:$PASS -XGET "https://$EP/products/_search" -H 'Content-Type: application/json' -d '{
  "size": 0,
  "aggs": {
    "by_tag":   { "terms": { "field": "tags" } },
    "by_month": { "date_histogram": { "field": "createdAt", "calendar_interval": "month" } }
  }
}'
```

## Step 4: Dashboards
> **Why:** OpenSearch Dashboards (Kibana fork) is the built-in UI. Port-forward from a bastion; create an index pattern on `products`; build a pie chart by `tags`. Zero code, stakeholder-friendly.

```bash
# From bastion with SSH tunnel
ssh -L 5601:$EP:443 ec2-user@bastion
# Browser: https://localhost:5601  (accept cert warning)
```

Dashboards → Stack Management → Index Patterns → create `products*` → Visualize → new pie chart on `tags.keyword`.

## Step 5: OpenSearch Serverless for vector search
> **Why:** RAG apps store embeddings (e.g. 1536-dim vectors from `text-embedding-3-small`) and query by cosine similarity. Serverless auto-scales; no node sizing. **Minimum 4 OCU (~$700/mo)** — destroy promptly.

```typescript
import * as aoss from 'aws-cdk-lib/aws-opensearchserverless';

const encPolicy = new aoss.CfnSecurityPolicy(this, 'Enc', {
  name: 'rag-enc',
  type: 'encryption',
  policy: JSON.stringify({
    Rules: [{ ResourceType: 'collection', Resource: ['collection/rag'] }],
    AWSOwnedKey: true,
  }),
});

new aoss.CfnCollection(this, 'Rag', {
  name: 'rag',
  type: 'VECTORSEARCH',
}).addDependency(encPolicy);
```

Create index with k-NN mapping:
```bash
curl -XPUT "https://$COLLECTION_EP/docs" -d '{
  "settings": { "index.knn": true },
  "mappings": {
    "properties": {
      "vec": { "type": "knn_vector", "dimension": 1536, "method": {"name":"hnsw","space_type":"cosinesimil"} },
      "text": { "type": "text" }
    }
  }
}'

# Query nearest neighbors
curl -XGET "https://$COLLECTION_EP/docs/_search" -d '{
  "size": 5,
  "query": { "knn": { "vec": { "vector": [...1536 floats...], "k": 5 } } }
}'
```

## Step 6: Fluent Bit → OpenSearch (container logs)
> **Why:** Observability pipeline. Fluent Bit sidecar on ECS/EKS tails container logs and ships to OpenSearch. Dashboards give you Kibana-style log search without paying Datadog.

```yaml
# fluent-bit.conf
[OUTPUT]
    Name            opensearch
    Match           *
    Host            ${OS_ENDPOINT}
    Port            443
    TLS             On
    AWS_Auth        On
    AWS_Region      us-east-1
    Index           logs
    Suppress_Type_Name On
```

## Step 7: Index State Management (ISM) — hot → warm → delete
> **Why:** Log indices grow forever. ISM policy auto-rolls them: today's index on hot nodes, 7-day old on UltraWarm (60% cheaper), 90-day old deleted. Mirrors S3 lifecycle for search data.

```bash
curl -u admin:$PASS -XPUT "https://$EP/_plugins/_ism/policies/logs-policy" -d '{
  "policy": {
    "description": "hot-warm-delete",
    "default_state": "hot",
    "states": [
      { "name":"hot",    "transitions":[{"state_name":"warm","conditions":{"min_index_age":"7d"}}] },
      { "name":"warm",   "actions":[{"warm_migration":{}}], "transitions":[{"state_name":"delete","conditions":{"min_index_age":"90d"}}] },
      { "name":"delete", "actions":[{"delete":{}}] }
    ],
    "ism_template": [{ "index_patterns": ["logs-*"] }]
  }
}'
```

## Cleanup
```bash
# Serverless collection first (ongoing OCU charges)
aws opensearchserverless delete-collection --id $COL_ID
cdk destroy   # ~10-15 min for managed domain delete
```

## Common Errors
- **Domain creation stuck 30+ min** — VPC subnet in wrong AZ / missing route. Check subnet count matches `zoneAwarenessConfig`.
- **`security_exception: no permissions for [indices:data/write/bulk]`** — fine-grained auth enabled but role not mapped to backend role in Dashboards → Security.
- **Cluster status RED** — shard unallocated, usually undersized storage. `GET _cluster/allocation/explain` reveals why.
- **k-NN slow / OOM** — HNSW index must fit in RAM. Rule of thumb: 1.1 × (num_vectors × dim × 4 bytes).
- **Serverless OCU bill shock** — min 4 OCU × $0.24 × 730 = ~$700/mo even idle.
- **Blue/green upgrade hours long** — normal for large domains; OpenSearch spins up parallel cluster and reindexes. Don't cancel.
