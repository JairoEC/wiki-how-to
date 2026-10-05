# INSTALACIÓN DE UN CLUSTER DE REDIS EN KUBERNETES

## Requisitos:
- Cluster de k8s xd

1. Crear el archivo redis-sts.yaml

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster
data:
  update-node.sh: |
    #!/bin/sh
    REDIS_NODES="/data/nodes.conf"
    sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${POD_IP}/" ${REDIS_NODES}
    exec "$@"
  redis.conf: |+
    cluster-enabled yes
    cluster-require-full-coverage no
    cluster-node-timeout 15000
    cluster-config-file /data/nodes.conf
    cluster-migration-barrier 1
    appendonly yes
    protected-mode no
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:8.8.2-alpine #VERSION DE LA IMAGEN DE REDIS
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

2. Crear redis-svc.yaml
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  selector:
    app: redis-cluster
```

3. Aplicar

```bash
kubectl apply -f redis-sts.yaml
#Salida:
#configmap/redis-cluster created
#statefulset.apps/redis-cluster created

kubectl apply -f redis-svc.yaml
#Salida:
#service/redis-cluster created
```

4. Ver los pods:

```
kubectl get pods -n redis-ns
<< 'COMMENT'
NAME                            READY   STATUS    RESTARTS   AGE
redis-cluster-0                 1/1     Running   0          49m
redis-cluster-1                 1/1     Running   0          49m
redis-cluster-2                 1/1     Running   0          49m
redis-cluster-3                 1/1     Running   0          49m
redis-cluster-4                 1/1     Running   0          49m
redis-cluster-5                 1/1     Running   0          49m
COMMENT
```


5. Incluir en el cluster

```bash
kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range .items[*]}{.status.podIP}:6379 {end}')
```

salida:
```
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.16.17.238:6379 to 172.16.191.99:6379
Adding replica 172.16.191.92:6379 to 172.16.17.205:6379
Adding replica 172.16.17.228:6379 to 172.16.191.71:6379
M: 6089ca3d943d7b3f0e32d2a5767e787a41fa9279 172.16.191.99:6379
   slots:[0-5460] (5461 slots) master
M: 4708d21e3d59529778d618d497eef3fdc05ef2ef 172.16.17.205:6379
   slots:[5461-10922] (5462 slots) master
M: 40328377518724ccf6d462ff97e2bae524dfc959 172.16.191.71:6379
   slots:[10923-16383] (5461 slots) master
S: 6cfcbc8501ae48d8f779db23b861791988330440 172.16.17.238:6379
   replicates 6089ca3d943d7b3f0e32d2a5767e787a41fa9279
S: 79fd6e1df96d965251f0e6432a8908bfe15261d2 172.16.191.92:6379
   replicates 4708d21e3d59529778d618d497eef3fdc05ef2ef
S: 02d3bbb228114667a96a9c889ce47dde5ec07779 172.16.17.228:6379
   replicates 40328377518724ccf6d462ff97e2bae524dfc959
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
......
>>> Performing Cluster Check (using node 172.16.191.99:6379)
M: 6089ca3d943d7b3f0e32d2a5767e787a41fa9279 172.16.191.99:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 4708d21e3d59529778d618d497eef3fdc05ef2ef 172.16.17.205:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 6cfcbc8501ae48d8f779db23b861791988330440 172.16.17.238:6379
   slots: (0 slots) slave
   replicates 6089ca3d943d7b3f0e32d2a5767e787a41fa9279
S: 79fd6e1df96d965251f0e6432a8908bfe15261d2 172.16.191.92:6379
   slots: (0 slots) slave
   replicates 4708d21e3d59529778d618d497eef3fdc05ef2ef
M: 40328377518724ccf6d462ff97e2bae524dfc959 172.16.191.71:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 02d3bbb228114667a96a9c889ce47dde5ec07779 172.16.17.228:6379
   slots: (0 slots) slave
   replicates 40328377518724ccf6d462ff97e2bae524dfc959
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```


6. Ingresar al cluster y probar agregando un "registro"

```
kubectl exec -it redis-cluster-0 -n redis-ns -- redis-cli -c
```
Salida:
```
127.0.0.1:6379> SET app2::llave "hola mundo 2"
-> Redirected to slot [15931] located at 172.16.191.108:6379
OK
172.16.191.108:6379> exit
```

7. Implementar redis insight:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redisinsight-service
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 5540
      nodePort: 30540 
  selector:
    app: redisinsight
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redisinsight-pv-claim
  labels:
    app: redisinsight
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redisinsight 
  labels:
    app: redisinsight
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: redisinsight 
  template: 
    metadata:
      labels:
        app: redisinsight 
    spec:
      volumes:
        - name: redisinsight
          persistentVolumeClaim:
            claimName: redisinsight-pv-claim
      initContainers:
        - name: init
          image: busybox
          command:
            - /bin/sh
            - '-c'
            - |
              chown -R 1000 /data
          resources: {}
          volumeMounts:
            - name: redisinsight
              mountPath: /data
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
      containers:
        - name:  redisinsight 
          image: redis/redisinsight:latest 
          imagePullPolicy: IfNotPresent 
          volumeMounts:
          - name: redisinsight
            mountPath: /data
          ports:
          - containerPort: 5540 
            protocol: TCP
```


Ahora aplicamos el manifiesto:

```bash
kubectl apply -f redisinsight.yaml
```