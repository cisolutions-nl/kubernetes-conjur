# Conjur - Kubernetes/OpenShift Integration with ESO

This document describes how to integrate [CyberArk Secrets Management](https://www.cyberark.com/products/secrets-management/) (Conjur) with Kubernetes or OpenShift. The integration
is done using the [External Secrets Operator (ESO)](https://external-secrets.io/latest/).
As the dummy application that reads secrets we use the image 
[nmatsui/hello-world-api](https://hub.docker.com/r/nmatsui/hello-world-api/).

## Prerequisites
To follow this guide, you need to have a Conjur instance
([Cloud](https://docs.cyberark.com/conjur-cloud/latest/en/Content/ConjurCloud/cl_ConjurCloudOverview.htm), 
[Enterprise](https://docs.cyberark.com/conjur-enterprise/latest/en/Content/Enterprise/WhatIsConjur.html?tocpath=Get%20started%7C_____1) 
or [Open Source](https://docs.cyberark.com/conjur-open-source/Latest/en/Content/Overview/Conjur-OSS-Suite-Overview.html)) running. 
And you need to have a Kubernetes cluster running. If you don't have one, you can use
[Minikube](https://minikube.sigs.k8s.io/docs/start/) or [k3s](https://k3s.io/) to
simulate a cluster locally or on your own server/VM.

The following tools (on your local machine) are required to follow this guide:
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/): Kubernetes CLI
- [helm](https://helm.sh/docs/intro/install/): Kubernetes package manager

In this demo we deploy the External Secrets Operator, Demo Application and the Reload in the **hello-world-eso** namespace.

## Installation
### Step 1: Install External Secrets Operator
First, we need to install the External Secrets Operator (ESO) in our Kubernetes cluster.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
    external-secrets/external-secrets \
    -n tools \
    --create-namespace \
    --set installCRDs=true
```

### Step 2: Create a Secret
To make sure Kubernetes can access Conjur, we need to create a secret with the Conjur
URL and the API key.

#### 2.1 Create a Conjur API Key
Create a host in conjur that can access to at least one secret. In this documentation we assume
that the host is named `host/testing/database-backup-script1` and has access to variable
`testing/database-password1`. And that the api key of this host is `api-key`.

#### 2.2 Convert the API Key and Host Id to Base64
Kubernetes Secrets are base64 encoded. So we need to convert the API key and host id to base64:

```bash
echo -n "testing/database-password1" | base64  # Output: dGVzdGluZy9kYXRhYmFzZS1wYXNzd29yZDE=
echo -n "api-key" | base64  # Output: YXBpLWtleQ==
```
#### 2.3 Create the Secret
Create a file ([secret.yml](kubernetes/secret.yml)) with the following content (replace the placeholders `<host_id_base64>`
and `<host_id_base64>` with the base64 encoded values):

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: conjur-creds
data:
  host_id: <host_id_base64>
  host_api_key: <host_id_base64>
```

and apply the secret to the cluster (for this demo we use the namespace `hello-world-eso`):

```bash
kubectl apply -f secret.yaml -n hello-world-eso
```

NB: Storing a secret in code is not recommended. In a production environment you should use a


### Step 3: Create an External Secret Store
Now we need to create an External Secret Store that points to the Conjur secret ([secret-store.yml](kubernetes/secret-store.yml)).
The External Secret Store is a custom resource that tells the External Secrets Operator how to
connect to the secret store (in this case Conjur).

```yaml
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: conjur
spec:
  provider:
    conjur:
      # Service URL
      url: https://conjur.pocpuppy.nl
      # [OPTIONAL] base64 encoded string of certificate
      # caBundle: <ca_bundle>
      auth:
        apikey:
          # conjur account
          account: admin
          userRef: # Get this from K8S secret
            name: conjur-creds
            key: host_id
          apiKeyRef: # Get this from K8S secret
            name: conjur-creds
            key: host_api_key
```

Apply the External Secret Store to the cluster:

```bash
kubectl apply -f secret-store.yml -n hello-world-eso
```

### Step 4: Create an External Secret
Now we can create an External Secret that points to the Conjur secret
([external-secret.yml](kubernetes/external-secret.yml)). The external
secret will map the Conjur secret to a Kubernetes secret.

```yaml
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: conjur
spec:
  refreshInterval: 10s
  secretStoreRef:
    # This name must match the metadata.name in the `SecretStore`
    name: conjur
    kind: SecretStore
  data:
    - secretKey: DB_PASSWORD1  # The key of the secret in Kubernetes
      remoteRef:
        key: testing/database-password1  # Reference to the Conjur secret
```

Apply the External Secret to the cluster:

```bash
kubectl apply -f external-secret.yml -n hello-world-eso
```

### Step 5: Create a Dummy Application
Now we can create a dummy application that reads the secret from Kubernetes
([hello-world.yml](kubernetes/hello-world.yml). We use the image `nmatsui/hello-world-api`
for this.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: nmatsui/hello-world-api:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          env:
            - name: DB_PASSWORD1  # The name of the environment variable in the container;
              valueFrom:
                secretKeyRef:
                  name: conjur
                  key: DB_PASSWORD1  # The key of the secret in Kubernetes;
```

Apply the dummy application to the cluster:

```bash
kubectl apply -f hello-world.yml -n hello-world-eso
```

## Testing
The following tests can be done to see if the integration works.

### Read the secret from Kubernetes
To test if the integration works, you can read the secret in the following way:

```bash
kubectl get secret -n hello-world-eso conjur -o jsonpath="{.data.DB_PASSWORD1}"  | base64 --decode && echo
```

This is a convenient way to see if the secret is available in Kubernetes and to check if
a secret changed after a rotation in Conjur.

### Read the secret in the dummy application
To see if the secret is available in the dummy application, you can do:

1. Get the pod name of the dummy application:
```bash
kubectl get pods -n hello-world-eso
```

Assume that the output contains:

```
NAME                                                READY   STATUS    RESTARTS   AGE
hello-world-5fd89f4c49-l2psr                        1/1     Running   0          42m
```

2. Read the secret from the container:
```bash
kubectl exec -it hello-world-5fd89f4c49-l2psr -n hello-world-eso -- env | grep DB_PASSWORD1
```

The output should be like `DB_PASSWORD1=<secret_value>`


## Reloading pods on secret change
The External Secrets Operator can automatically reload pods when a secret changes. To enable this
feature, you can use the package [Stakater](https://stakater.com/)'s
[Reloader](https://github.com/stakater/Reloader). This package watches for changes in secrets and
configmaps and reloads the pods that use them.

1. Install the Reloader package:
```bash
helm repo add stakater https://stakater.github.io/stakater-charts
helm install stakater/reloader -n tools  --set reloader.watchGlobally=false --generate-name
```

2. Add the annotation `reloader.stakater.com/auto: "true"` to the deployment of the dummy application
(`hello-world.yml`). This results in the following yaml file:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  annotations:  # Add this line
    reloader.stakater.com/auto: "true"  # Add this line
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: nmatsui/hello-world-api:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          env:
            - name: DB_PASSWORD1
              valueFrom:
                secretKeyRef:
                  name: conjur
                  key: DB_PASSWORD1
```

3. Apply the changes to the cluster:
```bash
kubectl apply -f hello-world.yml -n hello-world-eso
```
When you update a secret in Conjur, the pods will automatically
be recreated to reflect the change.

## Sources

The following sources have been used for this how-to:

https://docs.cyberark.com/conjur-open-source/Latest/en/Content/Integrations/k8s-ocp/k8s-jwt-secrets-store-eso.htm
https://external-secrets.io/latest/provider/conjur/
https://github.com/conjurdemos/Accelerator-K8s-External-Secrets