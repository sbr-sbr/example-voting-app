# Deployment Record

## Infrastructure

- **Cluster**: voting-app-cluster (EKS 1.31, us-east-1)
- **Nodes**: 3x t3.medium
- **VPC**: 10.0.0.0/16 across us-east-1a and us-east-1b
- **Provisioned with**: Terraform

## Helm Deployments

### Release 1 (Initial)
- Image: vote:latest
- Timestamp: Wed Oct 28 16:52:41 2026
- Status: ✅ Working

### Release 2 (Broken)
- Image: vote:broken-tag
- Timestamp: Wed Oct 28 17:38:33 2026
- Status: ❌ Failed (ImagePullBackOff - image tag did not exist in ECR)

### Release 3 (Rollback)
- Rolled back to Release 1
- Timestamp: Wed Oct 28 17:44:xx 2026
- Status: ✅ Working again

## Commands Executed

```bash
# Initial deploy
helm install voting ./helm/voting-chart --namespace default --create-namespace
# NAME: voting
# LAST DEPLOYED: Wed Oct 28 16:52:41 2026
# STATUS: deployed
# REVISION: 1

# Simulate bad deployment
helm upgrade voting ./helm/voting-chart \
  --set vote.image.tag=broken-tag \
  --namespace default
# Release "voting" has been upgraded.
# REVISION: 2

# Watch pods fail
kubectl get pods -w
# vote-864f759dd4-kqhvk    0/1     ImagePullBackOff   0          24s
# vote-864f759dd4-kqhvk    0/1     ErrImagePull       0          28s

# Check history
helm history voting
# REVISION  UPDATED                         STATUS      CHART               DESCRIPTION
# 1         Wed Oct 28 16:52:41 2026        superseded  voting-chart-1.0.0  Install complete
# 2         Wed Oct 28 17:38:33 2026        deployed    voting-chart-1.0.0  Upgrade complete

# Rollback to revision 1
helm rollback voting 1
# Rollback was a success! Happy Helming!

# Verify pods recovered
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# postgres-0               0/1     Pending   0          52m
# redis-66c6678858-q9dt9   1/1     Running   0          52m
# result-cf47fbff8-b47gh   1/1     Running   0          52m
# result-cf47fbff8-h776k   1/1     Running   0          52m
# vote-8f7cbc9b5-79pmq     1/1     Running   0          52m
# vote-8f7cbc9b5-986vv     1/1     Running   0          52m
# worker-77c669546-5c7pt   1/1     Running   0          52m
```

## Live Endpoints

- Vote app: http://a579324fe5c2e48978e9c7c51ee87000-402306058.us-east-1.elb.amazonaws.com:5000
- Result app: http://result-service-external-ip:5001

## Services

```
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)
kubernetes         ClusterIP      172.20.0.1       <none>                                                                   443/TCP
postgres-service   ClusterIP      172.20.212.240   <none>                                                                   5432/TCP
redis-service      ClusterIP      172.20.156.92    <none>                                                                   6379/TCP
result-service     LoadBalancer   172.20.227.15    <pending>                                                                5001/TCP
vote-service       LoadBalancer   172.20.13.142    a579324fe5c2e48978e9c7c51ee87000-402306058.us-east-1.elb.amazonaws.com   5000/TCP
```

## GitHub Actions CI/CD

- Build job: builds and pushes Docker images to ECR
- Trivy scan: scans images for HIGH/CRITICAL vulnerabilities
- Deploy job: updates EKS cluster via Helm

## Cleanup

```bash
helm uninstall voting
cd terraform/
terraform destroy -auto-approve
```
