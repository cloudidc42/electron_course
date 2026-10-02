# Part 54: Multi-Region & Disaster Recovery (Steps 591-610)

## Step 591: Multi-Region Architecture

```
Multi-Region Elixir Deployment:

Region: us-east-1 (Primary)          Region: ap-southeast-1 (Secondary)
┌────────────────────────────┐        ┌────────────────────────────┐
│ Load Balancer              │        │ Load Balancer              │
│  └── Phoenix Cluster (3)   │        │  └── Phoenix Cluster (3)   │
│       └── PostgreSQL       │◄──────►│       └── PostgreSQL       │
│            (Primary)       │  Repl  │          (Read Replica)    │
│       └── Redis Primary    │        │       └── Redis Replica    │
└────────────────────────────┘        └────────────────────────────┘
         │                                         │
         └──────────── Global CDN ─────────────────┘
                       Cloudflare / CloudFront

Traffic routing:
- Active-Active: both regions serve traffic (latency-based routing)
- Active-Passive: failover only when primary down
- Writes: always route to primary region
- Reads: serve from nearest replica
```

---

## Step 592: Database Replication

```elixir
# config/prod.exs - multi-region DB config
config :my_app, MyApp.Repo,
  url:       System.get_env("PRIMARY_DATABASE_URL"),
  pool_size: 20

config :my_app, MyApp.ReadRepo,
  url:       System.get_env("REPLICA_DATABASE_URL"),
  pool_size: 30,
  read_only: true

# Smart routing based on operation type
defmodule MyApp.SmartRepo do
  def get(schema, id, opts \\ []) do
    repo = if Keyword.get(opts, :primary), do: MyApp.Repo, else: MyApp.ReadRepo
    repo.get(schema, id, opts)
  end

  def all(query, opts \\ []) do
    repo = if Keyword.get(opts, :primary), do: MyApp.Repo, else: MyApp.ReadRepo
    repo.all(query, opts)
  end

  # Writes always go to primary
  def insert(changeset, opts \\ []), do: MyApp.Repo.insert(changeset, opts)
  def update(changeset, opts \\ []), do: MyApp.Repo.update(changeset, opts)
  def delete(record, opts \\ []),    do: MyApp.Repo.delete(record, opts)
  def transaction(fun, opts \\ []),  do: MyApp.Repo.transaction(fun, opts)
end

# Handle replication lag: after writes, use primary for subsequent reads
defmodule MyApp.ConsistencyGuard do
  def after_write(conn_or_socket, fun) do
    result = fun.()
    # Mark session to use primary for next N seconds
    put_in_session(conn_or_socket, :use_primary_until, unix_now() + 5)
    result
  end
end
```

---

## Step 593: Cross-Region Erlang Clustering

```elixir
# Erlang nodes across regions communicate via WAN
# Use libcluster with Kubernetes DNS or Consul

# config/prod.exs
config :libcluster,
  topologies: [
    # Same region: fast clustering via k8s DNS
    local: [
      strategy: Cluster.Strategy.Kubernetes.DNS,
      config: [
        service:          "myapp-headless.my-app.svc.cluster.local",
        application_name: "my_app"
      ]
    ],
    # Cross-region: via Consul or static config
    global: [
      strategy: Cluster.Strategy.Epmd,
      config: [
        hosts: [
          :"my_app@us-east-1.internal",
          :"my_app@ap-southeast-1.internal"
        ]
      ]
    ]
  ]

# Restrict expensive global operations to primary region
defmodule MyApp.RegionAware do
  def primary_region?, do: System.get_env("REGION") == "us-east-1"

  def run_on_primary(fun) do
    if primary_region?() do
      fun.()
    else
      # Forward to primary region via RPC
      :rpc.call(primary_node(), fun, [])
    end
  end

  defp primary_node do
    # Find a node in primary region
    Node.list()
    |> Enum.find(&String.contains?(to_string(&1), "us-east"))
    || node()
  end
end
```

---

## Step 594: Disaster Recovery Plan

```bash
#!/bin/bash
# disaster-recovery.sh - RTO: 15 minutes, RPO: 1 minute

echo "=== DISASTER RECOVERY PROCEDURE ==="

# 1. Check primary region health
echo "1. Checking primary region..."
if ! curl -sf https://us-east-1.myapp.com/health; then
  echo "PRIMARY DOWN - initiating failover"
  
  # 2. Promote read replica to primary
  echo "2. Promoting replica..."
  # AWS RDS:
  aws rds promote-read-replica \
    --db-instance-identifier myapp-replica-ap \
    --region ap-southeast-1
  
  # 3. Update DNS to point to secondary region
  echo "3. Updating DNS..."
  aws route53 change-resource-record-sets \
    --hosted-zone-id ZONE_ID \
    --change-batch file://failover-dns.json
  
  # 4. Scale up secondary region
  echo "4. Scaling secondary..."
  kubectl scale deployment my-app --replicas=6 --context=ap-southeast-1
  
  # 5. Notify team
  echo "5. Notifying team..."
  curl -X POST $SLACK_WEBHOOK \
    -d '{"text":"🚨 FAILOVER ACTIVATED: ap-southeast-1 is now primary"}'
fi
```

---

## Step 595: Backup Strategy

```bash
#!/bin/bash
# backup.sh - runs as Kubernetes CronJob

BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
S3_BUCKET="s3://myapp-backups"

# 1. PostgreSQL backup
echo "Backing up PostgreSQL..."
pg_dump $DATABASE_URL \
  --format=custom \
  --compress=9 \
  --no-password \
  -f /tmp/db_backup_$BACKUP_DATE.dump

# Encrypt backup
openssl enc -aes-256-cbc \
  -salt -pbkdf2 \
  -in  /tmp/db_backup_$BACKUP_DATE.dump \
  -out /tmp/db_backup_$BACKUP_DATE.dump.enc \
  -k   $BACKUP_ENCRYPTION_KEY

# Upload to S3 with lifecycle policy (90-day retention)
aws s3 cp /tmp/db_backup_$BACKUP_DATE.dump.enc \
  $S3_BUCKET/postgres/$BACKUP_DATE.dump.enc

# 2. Verify backup is restorable
echo "Verifying backup..."
pg_restore --list /tmp/db_backup_$BACKUP_DATE.dump > /dev/null && \
  echo "Backup verification: PASSED" || \
  echo "Backup verification: FAILED" | mail -s "BACKUP FAILED" ops@myapp.com

# Cleanup
rm /tmp/db_backup_*

echo "Backup completed: $BACKUP_DATE"
```

```yaml
# kubernetes/backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: my-app
spec:
  schedule: "0 */6 * * *"  # every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16-alpine
              command: ["/scripts/backup.sh"]
              envFrom:
                - secretRef:
                    name: backup-secrets
          restartPolicy: OnFailure
```

---

## Step 596: Health Checks

```elixir
defmodule MyAppWeb.HealthController do
  use MyAppWeb, :controller

  # Shallow check - just the app is alive
  def ping(conn, _params) do
    json(conn, %{status: "ok", timestamp: DateTime.utc_now()})
  end

  # Deep check - all dependencies healthy
  def health(conn, _params) do
    checks = %{
      database: check_database(),
      redis:    check_redis(),
      memory:   check_memory(),
      disk:     check_disk()
    }

    status = if Enum.all?(checks, fn {_, v} -> v.ok end), do: "healthy", else: "degraded"
    http_status = if status == "healthy", do: 200, else: 503

    conn
    |> put_status(http_status)
    |> json(%{status: status, checks: checks, node: node()})
  end

  defp check_database do
    try do
      MyApp.Repo.query!("SELECT 1")
      %{ok: true, latency_ms: measure_latency(fn -> MyApp.Repo.query!("SELECT 1") end)}
    rescue
      _ -> %{ok: false, error: "Database unreachable"}
    end
  end

  defp check_redis do
    case Redix.command(:redix, ["PING"]) do
      {:ok, "PONG"} -> %{ok: true}
      _             -> %{ok: false, error: "Redis unreachable"}
    end
  end

  defp check_memory do
    used = :erlang.memory(:total)
    limit = 500 * 1024 * 1024  # 500MB threshold
    %{ok: used < limit, used_mb: div(used, 1024 * 1024)}
  end

  defp check_disk do
    {output, 0} = System.cmd("df", ["-h", "/"])
    %{ok: true, output: output}
  end

  defp measure_latency(fun) do
    start = System.monotonic_time(:millisecond)
    fun.()
    System.monotonic_time(:millisecond) - start
  end
end

# router.ex
scope "/", MyAppWeb do
  get "/ping",   HealthController, :ping    # Kubernetes liveness
  get "/health", HealthController, :health  # Kubernetes readiness
end
```

---

## Step 597: Blue-Green Deployment

```yaml
# blue-green.yaml
# Run two identical deployments, switch traffic between them

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
  labels:
    app:     my-app
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app:     my-app
      version: blue
  template:
    metadata:
      labels:
        app:     my-app
        version: blue
    spec:
      containers:
        - name:  my-app
          image: my-app:1.0.0

---
# Service switches between blue and green
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app:     my-app
    version: blue    # ← change to 'green' to switch traffic
  ports:
    - port: 80
      targetPort: 4000
```

```bash
#!/bin/bash
# blue-green-switch.sh

CURRENT=$(kubectl get svc my-app -o jsonpath='{.spec.selector.version}')
NEW=$([ "$CURRENT" == "blue" ] && echo "green" || echo "blue")

echo "Switching from $CURRENT to $NEW..."

# Deploy new version to inactive slot
kubectl set image deployment/my-app-$NEW my-app=my-app:$NEW_VERSION
kubectl rollout status deployment/my-app-$NEW

# Run smoke tests against new slot
curl -f http://my-app-$NEW.internal/health || exit 1

# Switch traffic
kubectl patch svc my-app -p '{"spec":{"selector":{"version":"'$NEW'"}}}'
echo "Traffic switched to $NEW"
```

---

## Step 598: Canary Deployment

```yaml
# canary.yaml - route 10% of traffic to new version
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
    - my-app
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host:   my-app
            subset: canary
    - route:
        - destination:
            host:   my-app
            subset: stable
          weight: 90
        - destination:
            host:   my-app
            subset: canary
          weight: 10

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-app
spec:
  host: my-app
  subsets:
    - name:   stable
      labels:
        version: stable
    - name:   canary
      labels:
        version: canary
```

---

## Step 599: Incident Response

```markdown
# Incident Response Runbook

## Severity Levels
- P1: Complete outage, all users affected → 15-min response
- P2: Major degradation, >25% users affected → 1-hour response  
- P3: Minor issues → next business day

## P1 Response Steps
1. **Alert** - Page on-call engineer
2. **Triage** - Identify scope (all users? specific region? specific feature?)
3. **Communicate** - Status page update within 15 minutes
4. **Diagnose** - Check dashboards: Grafana → Jaeger → Logs
5. **Mitigate** - Fastest fix (rollback, disable feature flag, scale up)
6. **Resolve** - Fix root cause
7. **Postmortem** - Within 48 hours

## Common Issues & Fixes

### High Error Rate
```bash
# Check recent errors
kubectl logs -l app=my-app --tail=100 | grep ERROR

# Check DB connections
kubectl exec -it pod/postgres-0 -- psql -c "SELECT count(*) FROM pg_stat_activity"

# Scale up if CPU-bound
kubectl scale deployment my-app --replicas=6
```

### Memory Leak
```bash
# Find memory-hungry processes
kubectl top pods -n my-app

# Force GC via Elixir release
kubectl exec -it my-app-xxx -- bin/my_app rpc ":erlang.garbage_collect()"
```

### Database Slow
```bash
# Kill long-running queries
kubectl exec -it postgres-0 -- psql -c "
  SELECT pg_terminate_backend(pid) 
  FROM pg_stat_activity 
  WHERE state = 'active' AND now() - query_start > interval '5 minutes'
"
```
```

---

## Step 600: Global Load Balancing

```elixir
# Intelligent routing based on latency and capacity
defmodule MyApp.GlobalRouter do
  @regions [
    %{id: "us-east-1",      url: "https://us.myapp.com",  lat: 37.77},
    %{id: "ap-southeast-1", url: "https://ap.myapp.com",  lat: 1.35},
    %{id: "eu-west-1",      url: "https://eu.myapp.com",  lat: 51.51}
  ]

  def nearest_region(user_lat, user_lon) do
    @regions
    |> Enum.map(fn r ->
      Map.put(r, :distance, haversine(user_lat, user_lon, r.lat, r.lon))
    end)
    |> Enum.sort_by(& &1.distance)
    |> List.first()
  end

  defp haversine(lat1, lon1, lat2, lon2) do
    r = 6371  # Earth's radius in km
    dlat = (lat2 - lat1) * :math.pi / 180
    dlon = (lon2 - lon1) * :math.pi / 180
    
    a = :math.sin(dlat / 2) ** 2 +
        :math.cos(lat1 * :math.pi / 180) *
        :math.cos(lat2 * :math.pi / 180) *
        :math.sin(dlon / 2) ** 2
    
    c = 2 * :math.atan2(:math.sqrt(a), :math.sqrt(1 - a))
    r * c
  end
end
```

---

## สรุป Part 54

✅ **Step 591** - Architecture overview  
✅ **Step 592** - DB replication  
✅ **Step 593** - Cross-region clustering  
✅ **Step 594** - DR plan  
✅ **Step 595** - Backup strategy  
✅ **Step 596** - Health checks  
✅ **Step 597** - Blue-green deployment  
✅ **Step 598** - Canary deployment  
✅ **Step 599** - Incident response  
✅ **Step 600** - Global load balancing  

➡️ [Part 55: Advanced Testing Strategies](./part-55-testing-advanced.md)
