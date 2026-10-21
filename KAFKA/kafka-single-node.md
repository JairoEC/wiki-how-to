# INSTALACION DE KAFKA EN SINGLE NODE SOBRE KUBERNETES

## ***REQUISITOS***

fuente:
link | https://github.com/strimzi/strimzi-kafka-operator/releases/tag/1.2.0

### Resumen

Para la instalación de kafka, la forma mas sencilla es usar un operador; en este caso se recomienda usar strimzi-kafka como operador para facilitar el deploy de Kubernetes

#### ***PASOS***

1. Obtener el archivo yaml de config

> Se está usando la version v.1.2.0 para el strimzi operator, el cual tiene compatibilidad con las versiones de kafka: `kafka:v4.2.0`, `kafka:v4.2.1`, `kafka:v4.3.0`, `kafka:v4.3.1` y kubernetes en las versiones `v1.30 - v1.36`

```bash
wget https://strimzi.io/install/latest?namespace=kafka -O kafka-strimzi.yaml
```

2. Creacion de un namespace

```bash
kubectl create ns kafka
```

3. Aplicar el yaml del operator para el namespace `kafka`
```bash
kubectl apply -f kafka-strimzi.yaml -n kafka
```

4. (Opcional) Validar la instalacion del cdr de strimzi:
```bash
# Ver los crd
kubectl get crd | grep strimzi
```

Ejemplo de salida:
```bash
[root@k8s-master kafka]# kubectl get crd | grep strimzi
kafkabridges.kafka.strimzi.io                           2026-10-16T16:06:51Z
kafkaconnectors.kafka.strimzi.io                        2026-10-16T16:06:51Z
kafkaconnects.kafka.strimzi.io                          2026-10-16T16:06:49Z
kafkamirrormaker2s.kafka.strimzi.io                     2026-10-16T16:06:50Z
kafkanodepools.kafka.strimzi.io                         2026-10-16T16:06:50Z
kafkarebalances.kafka.strimzi.io                        2026-10-16T16:06:50Z
kafkas.kafka.strimzi.io                                 2026-10-16T16:06:50Z
kafkatopics.kafka.strimzi.io                            2026-10-16T16:06:50Z
kafkausers.kafka.strimzi.io                             2026-10-16T16:06:51Z
strimzipodsets.core.strimzi.io                          2026-10-16T16:06:49Z
```

```bash
# Validar el levantamiento del deploy de strimzi
kubectl wait deployment/strimzi-cluster-operator -n kafka --for=condition=Available --timeout=300s
kubectl describe deploy strimzi-cluster-operator -n kafka
# 
```

5. ***INSTALACIÓN DE KAFKA EN SINGLE NODE***

Primero obtengo el yaml del deploy de Kafka adaptado para correr con strimzi

```bash
wget https://strimzi.io/examples/latest/kafka/kafka-single-node.yaml -O kafka-deploy.yaml
## O el comando:

```

```bash
kubectl apply -f kafka-deploy -n kafka
```