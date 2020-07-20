
# Kubernetes deployment of ECR

The following instructions are targeted at a local test deployment via [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/). Everything is already pre-configured and should run as is. For a production deployment the configuration should be overwritten with kubernetes kustomize overlays.


## Preparation

```bash
minikube start

kubectl config use-context minikube

kubectl create namespace sage

# permanently save the namespace for all subsequent kubectl commands in that context.
kubectl config set-context --current --namespace=sage
```


## Deploy
```
minikube addons enable ingress

kubectl create configmap ecr-db-initdb-config -n sage --from-file=../schema.sql


# this starts all services, but Jenkins creates a token which the ecr-api will need
kubectl kustomize . | kubectl apply -f -
```

## Inject token

To let `ecr-api` talk to Jenkins a token is needed. Because Jenkins does not let us inject a token on startup, it is automatically generated when Jenkins starts. After Jenkins has started and generated a token for user `ecrdb`, the token has to be extracted form the Jenkins pod (container) and stored as a secret.

```

# get token from Jenkins or user ecrdb
export JENKINS_TOKEN=$(kubectl exec -ti $(kubectl get pods | grep "^ecr-jenkins-" | cut -f 1 -d ' ') -- /bin/cat /var/jenkins_home/secrets/ecrdb_token.txt)
echo "JENKINS_TOKEN=${JENKINS_TOKEN}"


# delete empty dummy secret
kubectl delete secret ecr-api-token-secret

# inject token as a secret
kubectl create secret generic ecr-api-token-secret --from-literal="token=${JENKINS_TOKEN}"

# restart api to use new token from secret
kubectl rollout restart deployment ecr-api

```

## IP address

In case of a minikube deployment:

```bash
minikube ip
```

Visit `<IP>/jenkins` to access the Jenkins instance.



# testing

```bash
kubectl exec -ti $(kubectl get pods | grep "^ecr-api-" | cut -f 1 -d ' ') -- /bin/ash -c 'coverage run -m pytest -v --runslow  &&  coverage report -m'
```