# Usage

```bash
# loads environment variables
$ source .envrc
```

### Requirements

Needs an IAM user with IAMOpenIdConnect Permissions

Get the node hostname with this command:

`kubectl get nodes -o jsonpath='{ $.items[*].status.addresses[?(@.type=="InternalDNS")].address }'`

Update the `local-storage.yaml` nodeSelector with the value from the previous command.

```
$ kubectl apply -f infra/k8s/local-storage.yaml

$ helm install postgresql bitnami/postgresql --set global.storageClass=local-storage

$ export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

$ echo $POSTGRES_PASSWORD

$ kubectl port-forward svc/postgresql 5432:5432

$ PGPASSWORD=$POSTGRES_PASSWORD psql --host 127.0.0.1 -U postgres -d postgres -p 5432

$ python -m venv env

$ source ./env/bin/activate

$ DB_USERNAME=username_here DB_PASSWORD=password_here python app.py

Add Token Class

Fix the api reports daily_usage function

```
