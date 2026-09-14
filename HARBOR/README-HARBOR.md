# INSTALAR HARBOR

La forma más sencilla y recomendada es realizar la instalación usando Helm

1. Descargar Harbor helm chart:

```bash
helm repo add harbor https://helm.goharbor.io
helm fetch harbor/harbor --untar
```

> Para este caso se hará la implementación de todos los recursos de harbor de forma interna en el cluster.

Creamos los certificados ***autofirmados*** para encriptar la comunicacion externa e interna.

```bash
openssl req -x509 -nodes -days 3650 -newkey rsa:4096   -keyout harbor.key -out harbor.crt   -subj "/CN=dominio.local"   -addext "subjectAltName=DNS:dominio.local"
```

Creamos un sercret con los certificados

```
kubectl create secret tls harbor-tls-secret --cert=harbor.crt --key=harbor.key -n harbor
```

```yaml
expose:
  type: route # Usar exposición por httproute-gateway api
  tls:
    enabled: false # Habilitar o deshabilitarla comunicacio https
    certSource: secret # Usamos el secret con las claves
.
.
.
    route:
        labels: {}
        annotations: {}
        parentRefs:
        - name: public
            namespace: nginx-gateway
        hosts:
        - "dominio.local"
.
.
.

externalURL: https://dominio.local


```