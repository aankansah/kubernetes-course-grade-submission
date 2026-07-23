NodePort ports  range - 30000 - 32000

By default the service type is ClusterIP if not specified

## Liveness Probes

check if the app running in the container is live. by default k8 autorestarts pods containers when they die but liveness probes lets us check the app running in the container as well. this continuously checks

## Readiness Probe

app could be operational but the services it depends on is not ready, so the readiness probe is used to check if the dependencies are ready before allowing track to reach the pod the app lives in. this continuously checks

## Stateful Set and Persistent Volume Claims (PVC)

- actual name generated for this pvc later will be mongodb-persistent-storage-mongodb-0, mongodb-persistent-storage-mongodb-1 (i.e. the pod name is appended to the pvc name)
- So the link is

  - POD -> PVC -> PERSISTENT VOLUME (actual storage on the node)
- Connection from the grade-submission-api to the mongodb statefulSet

  - GRADES API -> MONGODB SERVICE -> MONGODB STATEFULSET POD -> BOUND PERSISTENT VOLUME ON HOST
  - Full App Network Traffic Flow:
  - ```
    User/Portal
      -> grade-submission-api Service
      -> grade-submission-api Pod
      -> mongodb Service
      -> MongoDB StatefulSet Pod
      -> PVC
      -> bound PV
      -> storage backend
    ```



# ConfigMaps and Secrets

Never mix application-specific config with environment-related deployment details.

## Section 8: ConfigMaps and Secrets

### Why use them?

- ConfigMaps and Secrets let us move environment-specific values out of the container image and out of the Deployment manifest.
- This means the same app image can run in different environments with different configuration.
- The app should not need to be rebuilt just because a service hostname, port, username, or password changed.

### ConfigMaps

Use a ConfigMap for non-sensitive configuration.

Example from the grade-submission API:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grade-submission-api-config
  namespace: grade-submission
data:
  MONGODB_HOST: "mongodb"
  MONGODB_PORT: "27017"
```

The API uses these values to connect to MongoDB:

```text
MONGODB_HOST=mongodb
MONGODB_PORT=27017
```

`mongodb` works because it is the name of the MongoDB Service inside the same namespace.

Example from the portal:

```yaml
data:
  GRADE_SERVICE_HOST: grade-submission-api
```

The portal uses `grade-submission-api` because that is the Service name for the backend API.

### Secrets

Use a Secret for sensitive values like usernames, passwords, tokens, and keys.

Example from the grade-submission API:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grade-submission-api-secret
type: Opaque
data:
  MONGODB_USER: "YWRtaW4="
  MONGODB_PASSWORD: "cGFzc3dvcmQxMjM="
```

Secret values under `data` must be base64 encoded.

Examples:

```bash
echo -n 'admin' | base64
echo -n 'password123' | base64
```

Base64 is encoding, not encryption. It hides the plain text from the YAML, but anyone with permission to read the Secret can decode it.

`stringData` can be used to write plain text values in the manifest:

```yaml
stringData:
  SAMPLE_USER: "some_values"
```

Kubernetes converts `stringData` into base64 encoded `data` when it stores the Secret.

### Using ConfigMaps and Secrets in Deployments

The API Deployment loads all key/value pairs from both the ConfigMap and the Secret using `envFrom`:

```yaml
envFrom:
  - configMapRef:
      name: grade-submission-api-config
  - secretRef:
      name: grade-submission-api-secret
```

This creates environment variables inside the container:

```text
MONGODB_HOST
MONGODB_PORT
MONGODB_USER
MONGODB_PASSWORD
```

The portal Deployment loads its ConfigMap the same way:

```yaml
envFrom:
  - configMapRef:
      name: grade-submission-portal-config
```

### Namespace rule

ConfigMaps and Secrets must exist in the same namespace as the Pod that uses them.

For this project, the Deployments are in:

```text
grade-submission
```

So the ConfigMaps and Secrets should also be in:

```yaml
namespace: grade-submission
```

Important: `grade-submission-api-secret.yaml` should include the namespace too, otherwise it may be created in the current/default namespace instead of `grade-submission`.

### Relationship to Services

The ConfigMap does not create the connection by itself. It only gives the app the values it needs.

Actual API-to-MongoDB flow:

```text
grade-submission-api Pod
  -> reads MONGODB_HOST=mongodb from ConfigMap
  -> connects to mongodb Service on port 27017
  -> Service routes traffic to MongoDB StatefulSet Pod
  -> MongoDB writes data to /data/db
  -> /data/db is backed by the Pod's PVC/PV
```

### Apply order

Create ConfigMaps and Secrets before creating the Deployments that reference them.

Example:

```bash
kubectl apply -f grade-submission-api-config.yaml
kubectl apply -f grade-submission-api-secret.yaml
kubectl apply -f grade-submission-api-deployment.yaml
```

If a Pod references a missing ConfigMap or Secret, the Pod will not start correctly.

### Updating ConfigMaps and Secrets

- Updating a ConfigMap or Secret does not always automatically restart existing Pods.
- Environment variables are read when the container starts.
- If the app reads the value from env vars, restart the Deployment after changing the ConfigMap or Secret.

Example:

```bash
kubectl rollout restart deployment grade-submission-api -n grade-submission
```

### Useful commands

```bash
kubectl get configmaps -n grade-submission
kubectl get secrets -n grade-submission
kubectl describe configmap grade-submission-api-config -n grade-submission
kubectl describe secret grade-submission-api-secret -n grade-submission
kubectl get secret grade-submission-api-secret -n grade-submission -o yaml
```
