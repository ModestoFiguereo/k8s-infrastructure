# Deploy Gitlab
Manifests to deploy GitLab on Kubernetes

#### Bump versions

* gitlab-runner:alpine-v11.6.0
* gitlab:11.6.2-ce.0
* postgresql:9.5.15
* redis:5.0.3 (official redis container)

### Deploying GitLab itself
```
DOMAIN=<DOMAIN>                      # Assign your domain.

sed -i "s/<DOMAIN>/$DOMAIN/" ./gitlab/gitlab/gitlab-ingress.yaml

# create gitlab namespace
kubectl create -f ./gitlab/gitlab-namespace.yaml

# deploy redis
kubectl create -f ./gitlab/redis/redis-persistent-volume-claim.yaml
kubectl create -f ./gitlab/redis/redis-deployment.yaml
kubectl create -f ./gitlab/redis/redis-service.yaml

# deploy postgres
kubectl create -f ./gitlab/postgres/postgres-persistent-volume-claim.yaml
kubectl create -f ./gitlab/postgres/postgres-secret.yaml
kubectl create -f ./gitlab/postgres/postgres-config-map.yaml
kubectl create -f ./gitlab/postgres/postgres-deployment.yaml
kubectl create -f ./gitlab/postgres/postgres-service.yaml

# deploy gitlab itself
kubectl create -f ./gitlab/gitlab/gitlab-persistent-volume-claim.yaml
kubectl create -f ./gitlab/gitlab/gitlab-deployment.yaml
kubectl create -f ./gitlab/gitlab/gitlab-service.yaml
kubectl create -f ./gitlab/gitlab/gitlab-ingress.yaml
```

### Deploying GitLab Runner

```
# deploy Minio
kubectl create -f ./gitlab/minio/gitlab-persistent-volume-claim.yaml
kubectl create -f ./gitlab/minio/minio-deployment.yaml
kubectl create -f ./gitlab/minio/minio-service.yaml

# check that it's running
kubectl get pods --namespace=gitlab

    gitlab   minio-2719559383-y9hl2    1/1    Running   0   1m

# check logs for AccessKey and SecretKey
> $ kubectl logs -f minio-2719559383-y9hl2 --namespace=gitlab
+ exec app server /export

Endpoint:  http://10.2.23.7:9000  http://127.0.0.1:9000
AccessKey: 9HRGG3EK2DD0GP2HBB53
SecretKey: zr+tKa6fS4/3PhYKWekJs65tRh8pbVl07cQlJfkW
Region:    us-east-1

Browser Access:
   http://10.2.23.7:9000  http://127.0.0.1:9000

Command-line Access: https://docs.minio.io/docs/minio-client-quickstart-guide
   $ mc config host add myminio http://10.2.23.7:9000 9HRGG3EK2DD0GP2HBB53 zr+tKa6fS4/3PhYKWekJs65tRh8pbVl07cQlJfkW

Object API (Amazon S3 compatible):
   Go:         https://docs.minio.io/docs/golang-client-quickstart-guide
   Java:       https://docs.minio.io/docs/java-client-quickstart-guide
   Python:     https://docs.minio.io/docs/python-client-quickstart-guide
   JavaScript: https://docs.minio.io/docs/javascript-client-quickstart-guide


# create bucket named `runner`
> $ kubectl exec -it minio-2719559383-y9hl2 --namespace=gitlab -- bash -c 'mkdir /export/runner'

# register runner with token found on http://yourrunning-gitlab/admin/runners
> $ kubectl run -it runner-registrator --image=gitlab/gitlab-runner:v1.5.2 --restart=Never -- register
Waiting for pod default/runner-registrator-1573168835-tmlom to be running, status is Pending, pod ready: false

Hit enter for command prompt

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/ci):
http://gitlab.gitlab/ci
Please enter the gitlab-ci token for this runner:
_TBBy-zRLk7ac1jZfnPu
Please enter the gitlab-ci description for this runner:

Please enter the gitlab-ci tags for this runner (comma separated):
shared,specific
Registering runner... succeeded                     runner=_TBBy-zR
Please enter the executor: virtualbox, docker+machine, docker-ssh+machine, docker, docker-ssh, parallels, shell, ssh:
docker
Please enter the default Docker image (eg. ruby:2.1):
python:3.5.1
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
Session ended, resume using 'kubectl attach runner-registrator-1573168835-tmlom -c runner-registrator -i -t' command when the pod is running

# delete temporary registrator
> $ kubectl delete deployment/runner-registrator


```

Edit `gitlab/gitlab-runner/gitlab-runner-docker-configmap.yaml` and fill in your runner token and minio credentials.

```
# deploy configmap and runnner itself
> $ kubectl create -f ./gitlab-runner/gitlab-runner-docker-configmap.yaml
> $ kubectl create -f ./gitlab-runner/gitlab-runner-docker-deployment.yaml
```