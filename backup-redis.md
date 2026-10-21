# 1. Backup de system-database-single (PostgreSQL)
```
oc exec $(oc get pods -l 'deploymentconfig=system-database-single' -o json | jq -r '.items[0].metadata.name') -- sh -c 'pg_dump -U postgres bd-3scale-system' > system-database-backup.sql
```

```
gzip system-database-backup.sql
```

# 2. Backup de system-storage

```
mkdir -p ./local/dir
```
```
oc rsync $(oc get pods -l 'deployment=system-app' -o json | jq '.items[0].metadata.name' -r):/opt/system/public/system ./local/dir
```

# 3. Backup de backend-redis-single
```
oc cp $(oc get pods -l 'deploymentconfig=backend-redis-single' -o json | jq '.items[0].metadata.name' -r):/var/lib/redis/data/dump.rdb ./backend-redis-dump.rdb
```

# 4. Backup de system-redis-single
```
oc cp $(oc get pods -l 'deploymentconfig=system-redis-single' -o json | jq '.items[0].metadata.name' -r):/var/lib/redis/data/dump.rdb ./system-redis-dump.rdb
```

# 5. Backup de zync-database-single
```
oc exec $(oc get pods -l 'deploymentconfig=zync-database-single' -o json | jq -r '.items[0].metadata.name') -- sh -c 'pg_dump -U postgres bd-3scale-zync' > zync-database-backup.sql
```

```
gzip zync-database-backup.sql
```

# 6. Backup de OpenShift Secrets
```
oc get secrets system-smtp -o json > system-smtp.json
oc get secrets system-seed -o json > system-seed.json
oc get secrets system-database -o json > system-database.json
oc get secrets backend-internal-api -o json > backend-internal-api.json
oc get secrets system-events-hook -o json > system-events-hook.json
oc get secrets system-app -o json > system-app.json
oc get secrets system-recaptcha -o json > system-recaptcha.json
oc get secrets system-redis -o json > system-redis.json
oc get secrets zync -o json > zync.json
oc get secrets system-master-apicast -o json > system-master-apicast.json
```

# 7. Backup de ConfigMaps
```
oc get configmaps system-environment -o json > system-environment.json
```

```
oc get configmaps apicast-environment -o json > apicast-environment.json
```