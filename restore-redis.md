# Restore Completo Redis – 3scale (Backend + System)
Procedimiento detallado, comandos exactos y verificaciones finales

# 1. RESTORE: backend-redis
## 1.1 Identificar pods relacionados
oc get pods -n 3scale | grep backend-redis

## 1.2 Entrar al pod de restore
oc rsh -n 3scale backend-redis-single-restore-<ID>

## 1.3 Verificar rutas internas de Redis
ls -l /var/lib/redis
ls -l /var/lib/redis/data

## 1.4 Copiar el dump.rdb al pod
oc cp backend-redis-dump.rdb 3scale/backend-redis-single-restore-<ID>:/tmp/backend-redis-dump.rdb

## 1.5 Confirmar que llegó
ls -l /tmp

## 1.6 Reemplazar el dump
mv /tmp/backend-redis-dump.rdb /var/lib/redis/data/dump.rdb
chmod 660 /var/lib/redis/data/dump.rdb

## 1.7 Validar que Redis cargó el dump
redis-cli -a $REDIS_PASSWORD info persistence | grep load


Debe mostrar:

loading:0

## 1.8 Validar tamaño de DB
redis-cli -a $REDIS_PASSWORD dbsize

## 1.9 Ver claves restauradas
redis-cli -a $REDIS_PASSWORD keys '*' | head

## 1.10 Validar una clave específica
redis-cli -a $REDIS_PASSWORD get 'application/service_id:2/key:<ALGUNA_KEY>/id'

## 1.11 Validar persistencia completa
redis-cli -a $REDIS_PASSWORD info persistence

# 2. RESTORE: system-redis

El procedimiento es igual, solo cambia el pod y el archivo RDB.

## 2.1 Identificar el pod
oc get pods -n 3scale | grep system-redis


Ejemplo esperado:

system-redis-single-7-gbks7

## 2.2 Crear un segundo pod de restore (recomendado)

Si usas DeploymentConfig, como el backend, haz lo mismo:

Duplica el DC

Cámbiale el nombre: system-redis-single-restore

Apunta al nuevo PVC

Escala un solo pod

## 2.3 Entrar al pod de restore de system-redis
oc rsh -n 3scale system-redis-single-restore-<ID>

## 2.4 Confirmar rutas del contenedor
ls -l /var/lib/redis
ls -l /var/lib/redis/data

## 2.5 Copiar el dump RDB de system
oc cp system-redis-dump.rdb 3scale/system-redis-single-restore-<ID>:/tmp/system-redis-dump.rdb

## 2.6 Confirmar la copia
ls -l /tmp

## 2.7 Reemplazar el dump
mv /tmp/system-redis-dump.rdb /var/lib/redis/data/dump.rdb
chmod 660 /var/lib/redis/data/dump.rdb

## 2.8 Validar carga del dump
redis-cli -a $REDIS_PASSWORD info persistence | grep load

## 2.9 Validar tamaño de DB restaurada
redis-cli -a $REDIS_PASSWORD dbsize

## 2.10 Ver claves
redis-cli -a $REDIS_PASSWORD keys '*' | head

## 2.11 Validar persistencia completa
redis-cli -a $REDIS_PASSWORD info persistence