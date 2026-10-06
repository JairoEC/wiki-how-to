# Instalacion de GatewayApi Fabric Nginx
#cdr #nginx #gateway #httproute

[Enlace de referecia ](https://docs.nginx.com/nginx-gateway-fabric/install/manifests/open-source/#install-the-gateway-api-resources)

1. Instalación del CDR :

```bash
kubectl apply --server-side --force-conflicts -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/crds.yaml
```

2. Deberás de crear un namespace `nginx-gateway`, el cuál será usasdo por los manifiestos por defecto

```bash
kubectl create namespace nginx-gateway
```

3. Instalación del gateway fabric nginx:

```bash
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/default/deploy.yaml
```

4. Comprobar/verificar el deploy:

```bash
kubectl get pods -n nginx-gateway
```

<hr/>

# Configuración del recurso (Gateway api)

1. Creación del gateway class

  Solo si se necesita craer el recurso. Al instalar gateway, el recurso se crea

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
    name: nginx-gateway-class # nombre de ejemplo
spec:
    controllerName: gateway.nginx.org/gateway-controller # Nombre de ejemplo
```

2. Creación del Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: public
spec:
  gatewayClassName: nginx-gateway-class #indicar el gateway class
  listeners:
  - name: http-app
    protocol: HTTP #Protocolo
    port: 80 # Indicar puerto a exponer
    hostname: "*.example.local" #Especificar hostname / se indica * para aceptar todas las rutas que coincidan
    allowedRoutes:
      namespaces:
        from: All
```

3. Creación del HttpRoute

Ejemplo de httproute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-httproute # Indicar el nombre del objeto
  namepsace: app # Indicar un namespace
spec:
  parentRefs:
  - name: nginx-gateway # indicar al gateway
    namespace: public
  hostnames:
  - "app.example.local"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app-svc # Indicar el name del service
      port: 8080
```