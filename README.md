# Leafy Pay: MCK and OpenShift
Leafy Pay Repo:  
https://github.com/mongodb-industry-solutions/sec-fsi-pci-dss

## Steps:
- access the OpenShift account
- create a project (which is a namespace)
- install MCK
- deploy MCK
- deploy a replica-set

For this project, we will use the namespace `leafy-pay`


## Install MCK
To install MCK, the `Telemetry is disabled` since RBACs is required to be cluster-scoped and we don't have this access.  
Also, `runAsUser` is set to `null` to allow OpenShift to decide what user to use, since by default the Operator tries to use the user `2000` which is not allowed in OC for security reasons.

```
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update
 
helm install kubernetes-operator mongodb/mongodb-kubernetes \
   --namespace leafy-pay \
   --set operator.env=dev \
   --set operator.watchNamespace=leafy-pay \
   --set operator.telemetry.enabled=false \
   --set managedSecurityContext=true
```

## Deploy Ops Manager
Set the versions of OM and AppDB in ops-manager/deploy-om.yaml.
Then run the following command:

```
kubectl apply -f ./ops-manager/deploy-om.yaml
```

### Configure Ops Manager
Find the external IP of OM: with `k9s` go to services `:svc` and look for `ops-manager-svc-ext`.
Its IP is the one externally accessible. Access OpsManager via `http://<IP>:8080`.

Then:
- create an Organization with the name `leafy-org`
- create API Key in Access Manager (Organization Owner permissions)
- add the IP of the Operator pod to the access list
- update `config-map.yaml` and `replica-set-tls.yaml` with the orgId, private and public keys
- update the URL of OpsManager

## TLS

Create the CA:
```
kubectl apply -f certificate-authority.yaml
````

Create the CA Certificate:
```
kubectl get secret leafy-root-secret \
  -n cert-manager \
  -o jsonpath="{.data['ca\\.crt']}" | base64 --decode > ca.pem
```

Create the ConfigMap:
```
kubectl create configmap "replica-set-ca-configmap" \
  -n "leafy-pay" \
  --from-file=ca-pem=./ca.pem \
  --from-file=mms-ca.crt=./ca.pem \
  --from-file=ca.crt=./ca.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

Create the `leafy-root-secret`:
```
kubectl create secret generic leafy-root-secret \
  -n leafy-pay \
  --from-file=ca.crt=./ca.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

Issue the Certificates:
```
kubectl apply -f certificates.yaml
```

## Deploy the replica-set

Deploy replica-set with TLS enabled:
```
kubectl apply -f ../replica-set/replica-set-tls.yaml
```

## Create Database Users
To authenticate, we need to create database users:
```
kubectl apply -f admin.yaml -f user.yaml
```


### Building the connection string
Check the Kubernetes Services, and there will be 3 external IPs for each member of the RS.  
Use them and build the connection string.

```
mongodb://<external-host-0>:27017,<external-host-1>:27017,<external-host-2>:27017/?replicaSet=replica-set
```

## Observations for Automation

To expose the replica-set externally, two steps are needed and depend on each other:
1. External connectivity needs to be configured, in our case `LoadBalancer`
2. `horizons` must be configured, and it should use the hostnames of the LBs created above
3. issue the TLS certificates, including the LBs hostnames on it

So, horizons depends on the creation of the LoadBalancers, which means that to automate these steps, one the 2nd and 3rd step needs to wait the 1st step to be finished, and the LBs to be ready.

In summary, to automate `horizons` configuration:
```
create MongoDB CR with externalAccess (no horizons yet)
    → wait for N LoadBalancer Services to get external hostnames (poll oc/kubectl get svc)
    → patch CR with connectivity.replicaSetHorizons, ordered by member index
```

and to automate the TLS certs ussuing:
```
provision LBs → collect hostnames → generate/patch Certificate.dnsNames → wait for cert-manager to reissue → confirm CR/cert are in sync
```

### What real DBaaS platforms actually do instead (worth knowing):
- Atlas and most managed MongoDB-on-K8s platforms avoid per-member LBs entirely — they use a stable, pre-allocated DNS name per member (e.g., via Route53 + ExternalDNS operator) before provisioning the actual infrastructure, so the hostname is known upfront and gets baked into the cert and horizon config at creation time — no chicken-and-egg problem.
- Tools like ExternalDNS + a wildcard cert (*.mongodb.yourdomain.com) sidestep the "wait for LB, then patch cert" dance completely
— you decide the hostnames in advance, ExternalDNS points DNS records at whatever IP/LB gets created, and a single wildcard SAN covers all future members without per-member cert reissuance.
- This is also why MCK's externalAccess.externalDomain field exists — it's designed for exactly this pattern (predictable domain + wildcard cert), rather than the raw-ELB-hostname approach we used here out of necessity (no domain available).

**Conclusion:** scripting hostname-discovery-then-patch is a real requirement with this raw-LB approach, but a proper automated system would more likely invest in the
  ExternalDNS + wildcard-cert + externalDomain pattern specifically to eliminate that two-phase dependency and the recurring cert-SAN maintenance burden.