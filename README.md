# ESO on EKS demo

A small demo of [External Secrets Operator](https://external-secrets.io) (ESO) on Amazon EKS.
A sample service reads its config from AWS Secrets Manager through an `ExternalSecret`.
Change the secret in AWS and the value inside the running pod updates on its own.

- **Auth:** IRSA (IAM Roles for Service Accounts). No AWS keys are stored in the cluster.
- **Authorization:** the ESO IAM role can only read secrets under `<cluster>/*`.
- **Deploy:** kustomize (+ Helm via `helmCharts`) and `kubectl apply --server-side`.

## Layout

```
cluster/                         eksctl config, one file per cluster
aws/iam/<cluster>/               IAM trust + permission policies for ESO
k8s/platform/external-secrets/   ESO: base/ (shared) + <cluster>/ overlay
k8s/services/demo-app/           sample service: base/ + <cluster>/ overlay
```

Overlays are named after the cluster (`eso-demo-prod`; `eso-demo-stage` can be added the same way).

## Prerequisites

AWS CLI v2, eksctl, kubectl (within one minor version of the cluster; see `.tool-versions`),
Helm 3, kustomize 5, and an AWS profile with access to EKS, IAM and Secrets Manager.

```bash
export AWS_PROFILE=<profile> AWS_REGION=us-east-2 CLUSTER=eso-demo-prod
```

Replace `ACCOUNT_ID` placeholders with your AWS account ID before deploying.

## Quick start

```bash
# 1. Cluster (15–20 min). Creates the OIDC provider IRSA needs.
eksctl create cluster -f cluster/$CLUSTER.yaml

# 2. ESO IAM role. Put this cluster's OIDC issuer in the trust policy first:
aws eks describe-cluster --name $CLUSTER --query cluster.identity.oidc.issuer --output text
aws iam create-policy --policy-name k8s_${CLUSTER}_external_secrets \
  --policy-document file://aws/iam/$CLUSTER/external-secrets-policy.json
aws iam create-role --role-name k8s_${CLUSTER}_external_secrets \
  --assume-role-policy-document file://aws/iam/$CLUSTER/external-secrets-trust.json
aws iam attach-role-policy --role-name k8s_${CLUSTER}_external_secrets \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/k8s_${CLUSTER}_external_secrets

# 3. Secret in AWS (fake values only)
aws secretsmanager create-secret --name $CLUSTER/demo-app/config \
  --secret-string '{"GREETING":"hello from v1","API_TOKEN":"demo-token-v1"}'

# 4. ESO. Apply twice: the first pass creates the CRDs.
kustomize build --enable-helm --load-restrictor LoadRestrictionsNone \
  k8s/platform/external-secrets/$CLUSTER > /tmp/eso.yaml
kubectl apply --server-side -f /tmp/eso.yaml
kubectl wait --for condition=established crd/clustersecretstores.external-secrets.io
kubectl apply --server-side -f /tmp/eso.yaml
kubectl get clustersecretstore aws-secrets-manager      # READY True = IRSA works

# 5. Demo service
kustomize build k8s/services/demo-app/$CLUSTER | kubectl apply --server-side -f -
kubectl -n demo-app get externalsecret                  # SecretSynced
```

## The demo

```bash
kubectl -n demo-app logs deploy/demo-app -f              # terminal 1

aws secretsmanager put-secret-value --secret-id $CLUSTER/demo-app/config \
  --secret-string '{"GREETING":"hello from v2","API_TOKEN":"demo-token-v2"}'
kubectl -n demo-app annotate externalsecret demo-app force-sync=$(date +%s) --overwrite
```

The **file** value (`/etc/demo-app/GREETING`, mounted from the Secret) updates within a minute or two.
The **env var** keeps the old value until the pod restarts (`kubectl -n demo-app rollout restart deploy/demo-app`).

## Rebuilding after deleting the cluster

The IAM role and secrets survive a cluster delete. A new cluster gets a **new OIDC issuer**, so update
the trust policy before deploying ESO:

```bash
aws iam update-assume-role-policy --role-name k8s_${CLUSTER}_external_secrets \
  --policy-document file://aws/iam/$CLUSTER/external-secrets-trust.json
```

## Cleanup

Always delete with eksctl (a console delete leaves CloudFormation stacks and the OIDC provider behind):

```bash
eksctl delete cluster -f cluster/$CLUSTER.yaml --wait
```

IAM roles, policies and secrets are not removed by eksctl; delete them separately when you're done.
The cluster costs about $0.20/hour, so don't leave it running.
