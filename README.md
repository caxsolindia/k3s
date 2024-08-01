# K3S Path based routing with SSL

# Deployment Files

app1.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-react-app-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-react-app
      version: v1
  template:
    metadata:
      labels:
        app: my-react-app
        version: v1
    spec:
      containers:
      - name: my-react-app
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
      imagePullSecrets:
      - name: ecr-secret-ro

---

apiVersion: v1
kind: Service
metadata:
  name: my-react-app-v1
spec:
  selector:
    app: my-react-app
    version: v1
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

app2.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-react
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-react
      version: v1
  template:
    metadata:
      labels:
        app: my-react
        version: v1
    spec:
      containers:
      - name: my-react
        image: httpd
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: my-react
spec:
  selector:
    app: my-react
    version: v1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

## Ingress Configuration

ingress.yaml

```
# IngressRoute and Middleware for the first app: my-react-app-v1
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-react-app-v1-ingress
spec:
  entryPoints:
    - websecure
  routes:
    - match: PathPrefix(`/app1`)
      kind: Rule
      services:
        - name: my-react-app-v1
          port: 8080
      middlewares:
        - name: my-react-app-v1-stripprefix
  tls:
    secretName: nginx-selfsigned-cert
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: my-react-app-v1-stripprefix
spec:
  stripPrefix:
    prefixes:
      - /app1
---
# IngressRoute and Middleware for the second app: my-react
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-react-ingress
spec:
  entryPoints:
    - websecure
  routes:
    - match: PathPrefix(`/app2`)
      kind: Rule
      services:
        - name: my-react
          port: 80
      middlewares:
        - name: my-react-stripprefix
  tls:
    secretName: nginx-selfsigned-cert
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: my-react-stripprefix
spec:
  stripPrefix:
    prefixes:
      - /app2
```

# Create self signed certificate.

```
sudo apt install openssl
 
1. Generate a Private Key
 
openssl genpkey -algorithm RSA -out private.key -aes256
 
2. Generate a Certificate Signing Request (CSR)
 
openssl req -new -key private.key -out request.csr
 
3. Generate the Self-Signed Certificate
 
openssl x509 -req -days 365 -in request.csr -signkey private.key -out certificate.crt
 
4. Verify the Certificate
 
openssl x509 -in certificate.crt -text -noout
```

# Create secret for SSL:

```
openssl pkcs8 -topk8 -inform PEM -outform PEM -in /home/ubuntu/private.key -out /home/ubuntu/private-rsa.key -nocrypt
kubectl create secret tls nginx-selfsigned-cert --cert=/home/ubuntu/certificate.crt --key=/home/ubuntu/private.key -n default
```


