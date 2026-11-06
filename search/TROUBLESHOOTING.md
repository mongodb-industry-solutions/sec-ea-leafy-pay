# Troubleshooting MongoDB Search
Possible issues and fixes when deploying Search. Back to the [deploy steps](README.md).

### Search pods are not created (`0/3`, no pods) 
`FailedCreate ... unable to validate against any security context constraint`  
OpenShift rejects the default user/group `2000`. Make sure `search-tls.yaml` has the `securityContext` override (`runAsUser` and `fsGroup` = `1001350000`) for the three components:
- `clusters[0].statefulSet`
- `clusters[0].loadBalancer.managed.deployment`
- `observability.metricsForwarder.deployment`

The value must be inside the namespace range: `kubectl get ns leafy-pay -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'`

### `replica-set-search-0-0` stuck in `ContainerCreating`
`MountVolume.SetUp failed for volume "password" : secret "replica-set-search-sync-source-password" not found`  
The sync user secret was not created. Apply `user/search-sync.yaml`. The pod mounts the secret on the next kubelet retry (up to a few minutes). To retry right away:
```
kubectl delete pod replica-set-search-0-0 -n leafy-pay
```

### Search pod is `Running` but not ready (readiness probe 503)
The pod logs show `MongoSecurityException: Exception authenticating ... userName='search-sync-source'`.  
The `search-sync-source` user does not exist in the database. Check `kubectl get mongodbuser -n leafy-pay`. If it is `Failed`, see the next issue.

### `MongoDBUser` is `Failed` with `IP_ADDRESS_NOT_ON_ACCESS_LIST`
The operator cannot reach the Ops Manager API. Add the pod network to the Ops Manager access list (Organization -> Access List, or the API key access list).  
Use the cluster pod CIDR instead of a single pod IP, because the operator pod IP changes on restart:
```
kubectl get network.config cluster -o jsonpath='{.spec.clusterNetwork}'
```
(here `10.128.0.0/14`). The operator retries on its own. To force it:
```
kubectl rollout restart deployment/mongodb-kubernetes-operator -n leafy-pay
```

### `MongoDBSearch` stays `Pending` and the pods wait for a certificate secret
The `certsSecretPrefix` is `fts-vs`, so the secrets must be named:
- `fts-vs-replica-set-search-cert` (mongot server)
- `fts-vs-replica-set-search-lb-0-cert` (LB server)
- `fts-vs-replica-set-search-lb-0-client-cert` (LB client)

Check them with `kubectl get certificate,secret -n leafy-pay | grep fts-vs`. If a Certificate is not `Ready`, check that the issuer is `leafy-ca-issuer`: `kubectl describe certificate <name> -n leafy-pay`.

### Do not set both `certificateKeySecretRef` and `certsSecretPrefix`
`certificateKeySecretRef` is deprecated and wins when both are set, so the operator would look for the wrong secret. Use only `certsSecretPrefix`.

### Search pods fail TLS after the CA is renewed
The CA (`leafy-selfsigned-ca`) renews automatically, but `replica-set-ca-configmap` keeps the old CA until you update it. Refresh it from `leafy-root-secret` (see the TLS section in the [main README](../README.md)), and expect a rolling restart of the replica-set.
