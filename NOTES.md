NodePort ports  range - 30000 - 32000

By default the service type is ClusterIP if not specified

## Liveness Probes

check if the app running in the container is live. by default k8 autorestarts pods containers when they die but liveness probes lets us check the app running in the container as well. this continuously checks

## Readiness Probe

app could be operational but the services it depends on is not ready, so the readiness probe is used to check if the dependencies are ready before allowing track to reach the pod the app lives in. this continuously checks

## Stateful Set and Persistent Volume Claims (PVC)

- actual name generated for this pvc later will be mongodb-persistent-storage-mongodb-0, mongodb-persistent-storage-mongodb-1 (i.e. the pod name is appended to the pvc name)
