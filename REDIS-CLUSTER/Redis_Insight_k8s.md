## Deploy Redis Insight sobre K8S

Para el despliegue obtaremos por usar el manifiesto de la propia documentación, pero modificando el manifiesto del servicio porque optaremos por usar un **`NodePort`**.

La documentación te menciona que puedes hacer el despliegue de **Redis Insight** sin almacenamiento persistente, para esta prueba se contemplará usar un almacenamiento persistente:

Creamos el archivo:

```sh
vi redisinsight.yaml
```

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

El acceso a la consola web se ve de la siguiente manera:

![Consola_Web_Redis_Insight](Consola_web_Redis_Insight.png)

## Conectar Base de Datos con Redis Insight

Para realizar la conexión de una base de datos desplegada en **Kubernetes** con **Redis Insight** debes contemplar los siguientes valores:


| Campo    | Valor                           |
| -------- | ------------------------------- |
| Host     | Nombre del Servicio del pod     |
| Port     | 6379                            |
| Name     | Nombre del Pod                  |
| Username | vacío (si no configuraste auth) |
| Password | vacío (si no configuraste auth) |

Una vez tengas esos valores los ingresas en este módulo:

![Module_Add_DB_RI](Module_Add_DB.png)

Como paso final configuramos el modulo de logs de redis Insight. Para ello accedemos a la CLI del pod:

```bash
kubectl exec -it redis -- redis-cli
```

Dentro de la CLI ejecutamos este comando:

```bash
CONFIG GET slowlog-log-slower-than
```

La salida esperada debe ser:

```bash
1) "slowlog-log-slower-than"
2) "10000"
```

Aplicamos la modificación:

```bash
CONFIG SET slowlog-log-slower-than 0
```

Salida esperada:

```bash
OK
```

Con esta configuración, ahora cada consulta a la BD podrá ser vista por **Redis Insight**:

![Module_Logs_RI](RI_Logs.png)