# App Mesh x Flux x Flagger

This example shows how to integrate [AWS App Mesh](https://aws.amazon.com/app-mesh/) with [Weave Flux](https://docs.fluxcd.io) and [Flagger](https://docs.flagger.app/) from Weaveworks to implement service mesh, GitOps and canary deployment on Kubernetes.

## Setup

- Fork this repo. The repo will be updated automatically by Flux to reflect the latest deployment version.
- Add the missing values on the `.env` file. Specify the AWS account ID, region and the Slack webhook URL (this will be used by Flux to send the deployment progress to the Slack room).
- Edit the `./k8s/flux-values.yaml` and replace the `url:` value with the URL of the new repo forked from this repo.
- Run `./setup.sh` (make sure to connect to the right Kubernetes cluster), this will:
  - Provision the infrastructure ([Amazon EKS](https://aws.amazon.com/eks/) and [Amazon ECR](https://aws.amazon.com/ecr/)) with [AWS CDK](https://aws.amazon.com/cdk/).
  - Build the Docker image for the demo application and push it to ECR.
  - Install the tools including [AWS IAM Authenticator for Kubernetes](https://github.com/kubernetes-sigs/aws-iam-authenticator), [jq](https://stedolan.github.io/jq/), [Helm](https://helm.sh/) and Flux.
- Follow this [instruction](https://github.com/weaveworks/flux/blob/master/site/get-started.md#giving-write-access) to give Flux access to the repo.
- (Optional) X-Ray is used by default. However, Jaeger can be used instead by modifying the values in `releases/appmesh-inject.yaml`.

## Canary Release

- Run `fluxctl -n flagger list-workloads | grep flagger:deployment/backend`. You should see the deployments called `flagger:deployment/backend` and `flagger:deployment/backend-primary`, both are using version `1.0.0`. You cloud also run `kubectl -n flagger get all` to see the related Kubernetes resources being created.
- Run `fluxctl -n flagger list-images`. You should see that only the version `1.0.0` and being used (indicated by the pointing arrow).
- Run `kubectl -n flagger get pod`, make sure the `READY` column shows `3/3` (or `2/2` if Jaeger is used instead of X-Ray) for all the pods (this indicates App Mesh components were injected correctly). Otherwise, kill the pod(s) and wait for the new one to come up.
- Open another terminal,
  - Run `kubectl exec -it $(kubectl -n flagger get pod -l app=loadtester -o jsonpath='{.items[0].metadata.name}') -c loadtester bash` to get into the load tester pod.
  - Run `while true; curl api.flagger/test; echo ""; do sleep 1; done;`. You should see the version `1.0.0` on the responses.
- Run `./canary-release.sh`. This script will update the version of the application (on `app/package.json`), create a new commit and push to GitHub and then build the new Docker image and push to ECR.
- Check the behavior on terminal and App Mesh on AWS console.
  - Run `fluxctl -n flagger list-images` and `fluxctl -n flagger list-workloads | grep flagger:deployment/backend` to see that `flagger:deployment/backend` (the canary one) will be provisioned and deployed with the new version.
  - Flux will do the canary analysis and adjust App Mesh Virtual Router configuration (based on the values specified on the `canaryAnalysis` section of `flux/charts/backend/templates/canary.yaml`). You should see the mixed between version `1.0.0` and `2.0.0` from curl responses.
  - Finally, the primary one will move to the version `2.0.0`.
- Wait the process complete (Slack notification).
- Run `fluxctl -n flagger list-workloads | grep flagger:deployment/backend`. You should see that the deployments are using the new version of the image.
- Run `fluxctl -n flagger list-images`. You should see that the new image is being used.
- (Bonus) If you revert the Git commit or push the new commit with the tag on `flux/releases/backend.yaml` changed back to `1.0.0`. The canary deployment will be triggered again to move the deployment back.

## Tools

- Prometheus, run `kubectl port-forward $(kubectl get pod -l app.kubernetes.io/name=appmesh-prometheus -o jsonpath='{.items[0].metadata.name}' -n appmesh-system) 9090:9090 -n appmesh-system` and then open `http://localhost:9090`.
- Grafana, run `kubectl port-forward $(kubectl get pod -l app=flagger-grafana -o jsonpath='{.items[0].metadata.name}' -n appmesh-system) 3000:3000 -n appmesh-system` and then open `http://localhost:3000/d/flagger-appmesh/appmesh-canary?orgId=1&refresh=5s&var-namespace=flagger&var-primary=app-primary&var-canary=app&from=now-1h&to=now`.
- Jaeger, run `kubectl port-forward $(kubectl get pod -l app.kubernetes.io/name=appmesh-jaeger -o jsonpath='{.items[0].metadata.name}' -n appmesh-system) 16686:16686 -n appmesh-system` and then open `http://localhost:16686`.
