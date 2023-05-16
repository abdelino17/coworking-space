# Coworking Space Service Extension

The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

For this project, you are a DevOps engineer who will be collaborating with a team that is building an API for business analysts. The API provides business analysts basic analytics data on user activity in the service. The application they provide you functions as expected locally and you are expected to help build a pipeline to deploy it in Kubernetes.

## Deliverables

1. [`Dockerfile`](analytics/Dockerfile)
2. [Screenshot of AWS CodeBuild pipeline](screenshots/codebuild_pipeline.png)
3. [Screenshot of AWS ECR repository for the application's repository](screenshots/aws_ecr.png)
4. [Screenshot of `kubectl get svc`](screenshots/kubectl_svc.png)
5. [Screenshot of `kubectl get pods`](screenshots/kubectl_pods.png)
6. [Screenshot of `kubectl describe svc postgresql`](screenshots/kubectl_describe_svc_db.png)
7. [Screenshot of `kubectl describe svc analytics`](screenshots/kubectl_describe_svc_analytics.png)
8. [Screenshot of `kubectl describe deployment analytics`](screenshots/kubectl_describe_deploy_analytics.png)
9. [All Kubernetes config files used for deployment (ie YAML files)](analytics/k8s)
10. [Screenshot of AWS CloudWatch logs for the application](screenshots/analytics_logs.png)
11. `README.md`: You are reading it!

## Dependencies

### Local Environment

1. Python Environment - run Python 3.6+ applications and install Python dependencies via `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster
4. `helm` - apply Helm Charts to a Kubernetes cluster

### Remote Resources

1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. Kubernetes Environment with AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code

## Setup

In this paragraph, I will describe all the steps to deploy the project.

### 1. Bitnami Helm Repo

Set up the Bitnami Helm Repo.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### 2. Deploy the infrastructure

You can create the EKS Cluster via my [custom infra code](infra/terraform) written in [terraform](https://developer.hashicorp.com/terraform/downloads).

First, update [this line](https://github.com/abdelino17/coworking-space/blob/92f792bf94c311e8ba7907299a7c0f3aaf78a0ea/infra/terraform/main.tf#L297) with your repo and then deploy the infra with the following commands:

```bash
terraform init

terraform apply
```

This code will create the following:

- EKS Cluster
- ECR
- IAM Role
- CodeBuild

### 3. Install PostgreSQL Helm Chart

The [PostgreSQL Helm Chart](https://artifacthub.io/packages/helm/bitnami/postgresql) requires `aws-ebs-csi-driver` for persistent volume. If you are using your own AWS Account, you can uncomment the file [ebs-csi-driver.tf](infra/terraform/csi-driver.tf) to install it.

However, I noticed that the Udacity Account is missing the following permissions:

- iam:CreateOpenIDConnectProvider
- iam:ListOpenIDConnectProviders
- iam:GetOpenIDConnectProvider

Unfortunately without them, it was impossible to use EBS.

So I decided to use the `local-storage` Storage Class.

For that, you need to set the [node affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) first.

Get the node DNS with the command:

`$ kubectl get nodes -o jsonpath='{ $.items[*].status.addresses[?(@.type=="InternalDNS")].address }'`

Update the value in the [configuration](https://github.com/abdelino17/coworking-space/blob/92f792bf94c311e8ba7907299a7c0f3aaf78a0ea/infra/deployment/local-storage.yaml#L29). Then, deploy it!

`$ kubectl apply -f infra/deployment/local-storage.yaml`

Now you can install the postgresql with the `local-storage` storageclass.

```bash
$ helm install postgresql bitnami/postgresql --set global.storageClass=local-storage
```

This should set up a Postgres deployment at `postgresql.default.svc.cluster.local` in your Kubernetes cluster. You can verify it by running `kubectl svc`.

By default, it will create a username `postgres`. The password can be retrieved with the following command:

```bash
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

echo $POSTGRES_PASSWORD
```

### 4. Run Seed Files

We will need to run the seed files in `db/` in order to create the tables and populate them with data.

```bash
kubectl port-forward --namespace default svc/postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < 1_create_tables.sql

kubectl port-forward --namespace default svc/postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < 2_seed_users.sql

kubectl port-forward --namespace default svc/postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < 3_seed_tokens.sql
```

## Deployment

### 1. Running CodeBuild

To use CodeBuild, the easiest way is to add a `buildspec.yaml` file to your repo.

Replace the `AWS_ACCOUNT_ID` with yours.

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
      - cd analytics
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t analytics:latest .
      - docker tag analytics:latest AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/analytics:latest
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/analytics:latest
```

A build should be run if your configuration is right!

### 2. Analytics deployments

Update the [analytics application](analytics/deployment/deployment.yaml) config with your repo URL.

Then run these commands to deploy:

```bash
$ kubectl apply -f analytics/deployment/deployment.yaml

$ kubectl apply -f analytics/deployment/service.yaml
```

### 3. Verifying The Application

Get the application URL with this command:

```bash
export BASE_URL=$(kubectl get services analytics --output jsonpath='{.status.loadBalancer.ingress[0].hostname})'
```

- Generate report for check-ins grouped by dates
  `curl $BASE_URL/api/reports/daily_usage`

- Generate report for check-ins grouped by users
  `curl $BASE_URL/api/reports/user_visits`
