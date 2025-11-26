This lab walks through installing and configuring External Secrets Operator (ESO) on AKS, integrating it with Akeyless for secret sync.

Prerequisites
AKS cluster with kubectl configured and admin access.

Akeyless account with permissions to create secrets and Kubernetes Auth methods.

Familiarity with Helm for installing Kubernetes charts.

1) Install External Secrets Operator on AKS
```bash
kubectl create namespace external-secrets

helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets -n external-secrets
```
Verify installation:

```bash
kubectl get pods -n external-secrets
kubectl get crds | grep externalsecrets
```
2) Create a Kubernetes Auth Method in Akeyless
In Akeyless console or CLI:

Navigate to Authentication Methods → Create Kubernetes Auth.

Fill cluster API server URL, CA certificate, and bind to your access gateway.

Save the Auth Method name and ID.

3) Prepare AKS Service Account for Auth
Create a namespace and service account for ESO authentication.

```bash
kubectl create namespace akeyless-auth
kubectl create serviceaccount akeyless-auth-sa -n akeyless-auth
```
Extract JWT token, cluster CA, and API server URL:

```bash
SECRET_NAME=$(kubectl get sa akeyless-auth-sa -n akeyless-auth -o jsonpath='{.secrets[0].name}')
kubectl get secret $SECRET_NAME -n akeyless-auth -o jsonpath='{.data.token}' | base64 --decode > token.txt
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 --decode > ca.crt
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[0].cluster.server}' > apiserver.url
```
4) Create SecretStore in AKS
Create a Kubernetes Secret holding Akeyless Access ID, then create the SecretStore manifest.

Example akeyless-secretstore.yaml:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: akeyless-secret-store
  namespace: external-secrets
spec:
  provider:
    akeyless:
      apiURL: "https://api.akeyless.io"
      auth:
        kubernetes:
          serviceAccountRef:
            name: akeyless-auth-sa
            namespace: akeyless-auth
      accessId: "YOUR_AKEYLESS_ACCESS_ID"
```
Apply SecretStore:

```bash
kubectl apply -f akeyless-secretstore.yaml
```
5) Create ExternalSecret in Application Namespace
Create your application namespace if needed:

```bash
kubectl create namespace app-namespace
```
Create an ExternalSecret manifest (app-externalsecret.yaml):

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-db-secret
  namespace: app-namespace
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: akeyless-secret-store
    kind: SecretStore
  target:
    name: app-db-secret
    creationPolicy: Owner
  data:
    - secretKey: db-password
      remoteRef:
        key: /demo/app/db-password
```
Apply ExternalSecret:

```bash
kubectl apply -f app-externalsecret.yaml
```
6) Verify Secret Sync
Check ExternalSecret status:

```bash
kubectl describe externalsecret app-db-secret -n app-namespace
```
Check generated Kubernetes Secret:

```bash
kubectl get secret app-db-secret -n app-namespace -o yaml
kubectl get secret app-db-secret -n app-namespace -o jsonpath='{.data.db-password}' | base64 --decode
```
7) Deploy Application Pod Using Secret
Example Pod manifest (app-pod.yaml):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-app
  namespace: app-namespace
spec:
  containers:
    - name: app-container
      image: busybox
      command: ["sh", "-c", "echo DB_PASSWORD is $DB_PASSWORD && sleep 3600"]
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-db-secret
              key: db-password
```
Apply Pod:

```bash
kubectl apply -f app-pod.yaml
kubectl logs sample-app -n app-namespace
```
You should see the secret printed in logs.

8) Test Rotation
Update secret value in Akeyless web console or CLI at path /demo/app/db-password.

Wait for refreshInterval (1h) or update ExternalSecret spec to trigger immediate refresh.

Check app-db-secret in Kubernetes for updated value. Notice no Pod restart needed.

This completes your basic lab for AKS + Akeyless + External Secrets Operator.
