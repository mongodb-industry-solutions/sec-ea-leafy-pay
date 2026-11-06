# Deploy MongoDB Search (and Vector Search)
What we will do:
- Create the `search-sync-source` database user that Search uses to sync data from the replica-set.
- Deploy 3 Search (mongot) pods, 1 Envoy LoadBalancer and 1 metrics forwarder, with TLS.
- Issue the TLS certificates for Search with `cert-manager`.
- Import the `sample_mflix` sample data and create Search and Vector Search indexes to confirm everything works.

Please change the password in `user/search-sync.yaml` before applying the manifests.

## Prerequisites
- MCK 1.12 installed in the `leafy-pay` namespace
- Ops Manager deployed, and the operator IP/CIDR in its access list (see [TROUBLESHOOTING.md](TROUBLESHOOTING.md))
- Replica-set deployed with TLS and SCRAM authentication enabled
- TLS CA created (`tls/certificate-authority.yaml`) and the `leafy-ca-issuer` ClusterIssuer available

### What is deployed
| File | Resources |
|------|-----------|
| `user/search-sync.yaml` | `MongoDBUser` `search-sync-source-user` + the password `Secret` |
| `search/search-tls.yaml` | `MongoDBSearch` + 3 cert-manager `Certificate`s (mongot server, LB server, LB client) |

> The `MongoDBSearch` uses `certsSecretPrefix: fts-vs`, so the secret names must follow `fts-vs-replica-set-search-*`.
> The `securityContext` overrides (`1001350000`) in `search-tls.yaml` are needed on OpenShift. Do not remove them.

## Deploy Search

### 1. Create the sync user
Go to the repo root and run:
```
kubectl apply -f user/search-sync.yaml
```

Check that the user is `Updated`:
```
kubectl get mongodbuser -n leafy-pay
```

### 2. Deploy Search and the certificates
```
kubectl apply -f search/search-tls.yaml
```

## Verification

Open `k9s`, go to the pods (`:pods`) and check that all of these are `Running` and fully ready:

| Pod | Expected |
|-----|----------|
| `replica-set-search-0-0` | `1/1 Running` |
| `replica-set-search-0-1` | `1/1 Running` |
| `replica-set-search-0-2` | `1/1 Running` |
| `replica-set-search-lb-0-<hash>` | `1/1 Running` |
| `replica-set-search-metrics-forwarder-0-<hash>` | `1/1 Running` |

The 3 search pods start one at a time, so it can take a few minutes until all of them are up.

Also check in `k9s`:
- `:mongodbsearch` -> `replica-set` is in the `Running` phase
- `:mongodbusers` -> `search-sync-source-user` is `Updated`
- `:certificates` -> all certificates are `Ready`

## Import sample data

To confirm Search works, we import the `sample_mflix` sample database and create indexes on it.

### 1. Deploy the MongoDB Tools pod
Go to the repo root and run:
```
kubectl apply -f search/mongodb-tools.yaml
```
A `PodSecurity` warning can be ignored, the pod is still created. Check in `k9s` that `mongodb-tools-pod` is `Running`.

### 2. Copy the CA certificate to the pod
Generate `ca.pem` from the CA secret (skip if you already have it):
```
kubectl get secret leafy-root-secret \
  -n cert-manager \
  -o jsonpath="{.data['ca\\.crt']}" | base64 --decode > ca.pem
```

Copy it to the pod:
```
kubectl cp ./ca.pem leafy-pay/mongodb-tools-pod:/tmp/ca.pem
```

### 3. Download the sample data
Open a shell in the pod (in `k9s`, select `mongodb-tools-pod` and press `s`), or run:
```
kubectl exec -n leafy-pay -it mongodb-tools-pod -- sh
```

Inside the pod, download the sample data:
```
curl -fSL https://atlas-education.s3.amazonaws.com/sample_mflix.archive -o /tmp/sample_mflix.archive
```

### 4. Restore the data
Still inside the pod. Replace `<password>` with the `mdb-admin` password:
```
mongorestore \
  --uri="mongodb://mdb-admin:<password>@replica-set-svc.leafy-pay.svc.cluster.local:27017/?replicaSet=replica-set&authSource=admin&tls=true&tlsCAFile=/tmp/ca.pem" \
  --archive=/tmp/sample_mflix.archive \
  --nsInclude 'sample_mflix.*'
```

The password is in the `mdb-admin-user-password` secret:
```
kubectl get secret mdb-admin-user-password -n leafy-pay -o jsonpath='{.data.password}' | base64 --decode
```

## Create Search indexes

### 1. Connect to the replica-set
Inside the pod, connect to the replica-set with `mongosh` (TLS and the `mdb-admin` user):
```
mongosh \
  --username mdb-admin \
  --password <password> \
  --authenticationDatabase admin \
  --tls \
  --tlsCAFile /tmp/ca.pem \
  "mongodb://replica-set-0.replica-set-svc.leafy-pay.svc.cluster.local:27017,replica-set-1.replica-set-svc.leafy-pay.svc.cluster.local:27017,replica-set-2.replica-set-svc.leafy-pay.svc.cluster.local:27017/?replicaSet=replica-set"
```

Use the `sample_mflix` database:
```
use sample_mflix
```

### 2. Create a Vector Search index
In the `embedded_movies` collection:
```
db.embedded_movies.createSearchIndex("vector_index", "vectorSearch",
    { "fields": [ {
      "type": "vector",
      "path": "plot_embedding_voyage_3_large",
      "numDimensions": 2048,
      "similarity": "dotProduct",
      "quantization": "scalar"
    } ] });
```

### 3. Create a Search index
In the `movies` collection:
```
db.movies.createSearchIndex("default", { mappings: { dynamic: true } });
```

## Verify Search is working

List the indexes. The `status` of each one must become `READY` (building can take a few minutes):
```
db.runCommand({"listSearchIndexes": "movies"});
db.runCommand({"listSearchIndexes": "embedded_movies"});
```

Optionally, run a quick search query to confirm results come back:
```
db.movies.aggregate([
  { $search: { index: "default", text: { query: "baseball", path: "title" } } },
  { $limit: 3 },
  { $project: { _id: 0, title: 1 } }
]);
```

If the indexes stay in `PENDING` or the query fails, check [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## Possible issues
If something does not work, check [TROUBLESHOOTING.md](TROUBLESHOOTING.md).
