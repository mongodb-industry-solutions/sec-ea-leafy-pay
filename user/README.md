# Enable SCRUM and create MongoDB database users
What we will do:
- Enable SCRUM authentication in the replica-set deployment
- Create 2 users to secure the access to the database.
- Cretae 2 Kubernetes secrets with the user passwords. 

Please change the passwords before applying the manifests. 

### The users are:

- `mdb-admin` have full access to all databases.
- `mdb-user` read and write access only to the `sample_mflix` database, for Vector Search.

### Create the database users
Go to the folder `./user/` and run:
```
kubectl apply -f admin.yaml -f user.yaml
```

## Verification

You can check in OpsManager if all users were created properly: `Deployment -> replica-set -> Security`

