# Vault in Kubernetes Tutorial

This is a proof of concept [Vault](https://www.vaultproject.io/) with [Raft HA](https://learn.hashicorp.com/vault/operations/raft-storage) in an AWS [Kubernetes](https://kubernetes.io/) cluster created using [`eksctl`](eksctl.io) using the [vault-operator](https://operatorhub.io/operator/vault) by Banzai Cloud. [Helm](https://hub.helm.sh/) is used to deploy the dependencies to the cluster.

## Kubernetes Primer

Kubernetes is a container orchestration standard contributed to by all the major cloud providers which provides a standard way to describe deployment of various applications. Primarily the manifests are written in YAML, but to avoid repetition often times other tools are used that provided less-than turing complete programmability. Before we talk about that though, lets go over the building blocks of a Kubernetes application.

### Resources

Every atomic unit Kubernetes is called a `Resource`. When you run `kubectl get pods -n default`, you are getting all the resource of type `Pod` in the default namespace. There are many different types of resources for various purposes. `Service` is for load balancing to pods, `Deployment` is for replicating a stateless service, `DaemonSet` for running a daemon on all nodes in the cluster, etc.

All the above resources are *namespaced* resources, meaning they are grouped together in a `Namespace`. `Namespace`s are extremely useful because they provide a software level of isolation for role based permissioning and make it easy to organize and group relevent resources together. For example, all the Kubernetes resources necessary for running and monitoring our vault cluster will live in one namespace, whereas the operators will live in their own separate namespaces. More on operators later.

Next lets see how to nicely package these resources together.

### Helm

[Helm](https://helm.sh/) is the prominent package manager for Kubernetes, which has a [large community of contributors](https://github.com/helm/helm/graphs/contributors) and now has over 800+ charts available for install on the [Helm Hub](https://hub.helm.sh/). It's basically YAML with [moustache templates](https://en.wikipedia.org/wiki/Mustache_(template_system)) customized with a light programming language packaged in reusable modules that can be configured for your needs. There are three upstream helm charts that are used in this project:
* [Vault Operator](https://hub.helm.sh/charts/banzaicloud-stable/vault-operator) by Banzai Cloud
* [Prometheus Operator](https://hub.helm.sh/charts/bitnami/prometheus-operator) by CoreOS

You are probably wondering what Operators are now. Lets go over that next.

### Operators

A key concept in Kubernetes are [Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/), which are operational knowledge of an application packages in a controller. They create new resources with [CustomResourceDefinitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions) like `Vault` and `Prometheus` to make it easier to deploy production ready versions of these services, while only caring about the configuration you need at a high level.

For example, when one deploys a `Vault`, the operator gets notified of this event, then starts to create the other Kubernetes resources it needs to run that Vault. In this case, it needs to create a `Secret` for the unseal keys, `StatefulSet` for running the vault containers, and `Service`s to load balance to the vault containers.

Since all Operators interact with the Kubernetes API, they are can be used together. For example a `Vault` YAML file can specify `serviceMonitor: enabled` to automatically create a `ServiceMonitor` CRD which tells the Prometheus Operator to start scraping vault for prometheus metrics.

## Tutorial

### Install required components

Follow the instructions for your platform:
* [eksctl](https://eksctl.io/introduction/#installation)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [helm](https://helm.sh/docs/intro/install/)

*Optional*: If you want to see resources in a UI install [octant](https://octant.dev/docs/master/index.html#installation).

### Launch the EKS cluster

In your AWS account make sure there is not an existing EKS cluster named `vault`. You can change the name of the cluster in [`cluster.yaml`](./cluster.yaml). Then run:

```bash
eksctl create cluster -f cluster.yaml
```

Wait for the cluster to come up, then verify you can access the cluster by listing the system pods:

```bash
kubectl get pods -n kube-system
# NAME                       READY   STATUS    RESTARTS   AGE
# aws-node-bzz9p             1/1     Running   0          6m40s
# aws-node-mvvd8             1/1     Running   0          6m40s
# aws-node-plw8s             1/1     Running   0          6m44s
# coredns-5c97f79574-8bpck   1/1     Running   0          14m
# coredns-5c97f79574-csmz7   1/1     Running   0          14m
# kube-proxy-2xm6x           1/1     Running   0          6m40s
# kube-proxy-5vrxh           1/1     Running   0          6m44s
# kube-proxy-nblns           1/1     Running   0          6m40s
```

Or to view the cluster resources in a UI run:

```bash
octant
```

This will open your browser and uses your local `kubeconfig` to authenticate and query the Kubernetes API.

### Install the Operators

Now its time to use `helm` to install the operators.

#### Vault Operator

To install the [Vault Operator](https://hub.helm.sh/charts/banzaicloud-stable/vault-operator) first we need to add the helm repository.
```bash
helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com # Adds a remote helm repository
# "banzaicloud-stable" has been added to your repositories
```

Then let's create a namespace to install the chart in and switch to that namespace:
```bash
kubectl create ns vault-operator
# namespace/vault-operator created
kubectl config set-context --current --namespace=vault-operator
# Context "kbambridge@vault.us-west-2.eksctl.io" modified.
```

Now if you run `kubectl get pods` you'll see that nothing will show up.

Next, install the chart:
```bash
helm install vault-operator banzaicloud-stable/vault-operator
# NAME: vault-operator
# LAST DEPLOYED: Sun May 24 13:31:55 2020
# NAMESPACE: vault-operator
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
```

Now you should see the `vault-operator` pod running:
```bash
kubectl get pods
# NAME                              READY   STATUS    RESTARTS   AGE
# vault-operator-65457bf776-9gshc   1/1     Running   0          23s
```

And you should see a new `vault` CRD:
```bash
kubectl get crd | grep vault
# vaults.vault.banzaicloud.com            2020-05-24T20:31:52Z
```

Now we can create `Vault`s!

### Create a Vault with Raft HA

Let's create another namespace for deploying our vault:
```bash
kubectl create ns vault
# namespace/vault created
kubectl config set-context --current --namespace=vault
# Context "kbambridge@vault.us-west-2.eksctl.io" modified.
```

Create the `Vault` by applying the kubernetes manifest in `./k8s/vault-raft.yaml`:
```bash
kubectl apply -f ./k8s/vault-raft.yaml
# vault.vault.banzaicloud.com/vault created
```

Observe that the vault was created:
```bash
kubectl get vault
# NAME    AGE
# vault   3s
```

Now lets see the vault pods:
```bash
kubectl get pods
# No resources found in vault namespace.
```

Wait a minute... what happened here? Lets take a look at the logs in the operator:

```bash
kubectl logs -l app.kubernetes.io/name=vault-operator -n vault-operator
# {"level":"info","ts":1590352663.9628658,"logger":"controller_vault","msg":"Reconciling Vault","Request.Namespace":"vault","Request.Name":"vault"}
# {"level":"info","ts":1590352664.2337954,"logger":"controller_vault","msg":"TLS CA will be regenerated due to: ","error":"an empty CA was provided"}
# {"level":"info","ts":1590352713.6105566,"logger":"controller_vault","msg":"Reconciling Vault","Request.Namespace":"vault","Request.Name":"vault"}
```

Note that we had to specify `-n vault-operator` since the operator is in a different namespace than our current namespace `vault`.

Doesn't seem like any errors occured in the operator so lets see if there are any events in the `vault` namespace:
```bash
kubectl get events
# LAST SEEN   TYPE      REASON                 OBJECT                                     MESSAGE
# 112s        Warning   FailedCreate           replicaset/vault-configurer-6c459f7f66     Error creating: pods "vault-configurer-6c459f7f66-" is forbidden: error looking up service account vault/vault: serviceaccount "vault" not found
# 12m         Normal    ScalingReplicaSet      deployment/vault-configurer                Scaled up replica set vault-configurer-6c459f7f66 to 1
# 2m45s       Normal    WaitForFirstConsumer   persistentvolumeclaim/vault-raft-vault-0   waiting for first consumer to be created before binding
# 13m         Normal    EnsuringLoadBalancer   service/vault                              Ensuring load balancer
# 13m         Normal    EnsuredLoadBalancer    service/vault                              Ensured load balancer
# 12m         Normal    SuccessfulCreate       statefulset/vault                          create Claim vault-raft-vault-0 Pod vault-0 in StatefulSet vault success
# 113s        Warning   FailedCreate           statefulset/vault                          create Pod vault-0 in StatefulSet vault failed error: pods "vault-0" is forbidden: error looking up service account vault/vault: serviceaccount "vault" not found
```

Note `error looking up service account vault/vault: serviceaccount "vault" not found`. Looks like we forgot to create the `ServiceAccount` and give it permission to access secrets! Create the [RBAC (Role Based Access Control)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) resources by running:
```bash
kubectl apply -f ./k8s/rbac.yaml
# serviceaccount/vault created
# role.rbac.authorization.k8s.io/vault-secrets created
# rolebinding.rbac.authorization.k8s.io/vault-secrets created
```

Now wait until the `StatefulSet` and `ReplicaSet` tries to create pods again (it has an exponential backoff so it may depend how long you took to apply the RBAC).
```bash
watch kubectl get pods
# NAME                                READY   STATUS    RESTARTS   AGE
# vault-0                             1/2     Running   0          14s
# vault-configurer-58994fd55f-jssps   1/1     Running   0          37s
```

> *Note*: If you want to speed it up you can delete the `Vault` and recreate it:
> ```bash
> kubectl delete -f ./k8s/vault-raft.yaml && kubectl apply -f ./k8s/vault-raft.yaml
> # vault.vault.banzaicloud.com "vault" deleted
> # vault.vault.banzaicloud.com/vault created
> ```

The first vault instance is being provisioned and configured, the other ones will come up sequentially. The `StatefulSet` and `Deployment` controllers in Kubernetes gradually scale them up as Kubernetes knows they are healthy. This is implemented with [health checks](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

Make sure all the pods are up:
```bash
kubectl get pods
# NAME                                READY   STATUS    RESTARTS   AGE
# vault-0                             2/2     Running   0          2m2s
# vault-1                             2/2     Running   0          102s
# vault-2                             2/2     Running   0          78s
# vault-configurer-58994fd55f-jssps   1/1     Running   0          2m25s
```

The `2/2` means that 2 out of the 2 desired containers are running and healthy. Things are looking good!

Now how can we get access to this cluster? Lets take a look at the [`Service`](https://kubernetes.io/docs/concepts/services-networking/service/):

```bash
kubectl get service
# NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
# vault              ClusterIP   10.100.100.7    <none>        8200/TCP,8201/TCP,9091/TCP,9102/TCP   24m
# vault-0            ClusterIP   10.100.158.97   <none>        8200/TCP,8201/TCP,9091/TCP            25m
# vault-1            ClusterIP   10.100.4.65     <none>        8200/TCP,8201/TCP,9091/TCP            25m
# vault-2            ClusterIP   10.100.61.238   <none>        8200/TCP,8201/TCP,9091/TCP            25m
# vault-configurer   ClusterIP   10.100.13.98    <none>        9091/TCP                              24m
```

We can port-forward to localhost to get access to the cluster:

```bash
kubectl port-forward svc/vault 8200
# Forwarding from 127.0.0.1:8200 -> 8200
# Forwarding from [::1]:8200 -> 8200
```

Now go to https://localhost:8200/ui/vault/auth. You can notice by the green dot at the top right that the Vault is unsealed. But where are the unseal tokens?

They are stored in the kubernetes secret `vault-unseal-keys`:
```bash
kubectl get secret -oyaml vault-unseal-keys
# ...
# data:
#   vault-root: cy44SDZZS1BLNE9Hb1o0bWVOdm9KTE1nZG4=
#   vault-test: dmF1bHQtdGVzdA==
#   vault-unseal-0: OGYwOGNkNDMyNzVmMGZjNWI2MWM1Nzc4Mjg1M2IwN2VkN2Y1ZmJkM2Y4NjBjODZlZmZiMmMyYWNmNmZjMTcwYTNm
#   vault-unseal-1: MDMxNTM3MjllN2JiNTE2OWM3OTUxZWVhNmQ2OWU3MWUyMjFlYzU0OWI3YmM2MzI5MjQ3MjQ3YTYwN2VlZDM2MjE4
#   vault-unseal-2: MmJkMDlhMzA4MTE5YmUzZWFiOGU5MzE5NjgxNDgyYTcxMjI5MGMwNTA3NzAxNGNiYjMxM2Q0MjU2MTk4MDljZTE3
#   vault-unseal-3: N2UzOWE1YzA0NDczMzU1ZDRhZmZjMGUwZDMzMDViZTliMjBhZThmZjgyM2U1M2ZjYzM1OTRkMGFlOTllZGE1NzU4
#   vault-unseal-4: ODFiMjcyOTQ5ZjIzNTdiODUyYmE4Y2VlYmU3YWJhZjY1MzY3ZDY3MjE4NmY0NzgwNmM2YzI5MGJmY2M4N2UxOGZk
# ...
```
They are base64 encoded so you can copy and paste the vault-root token and convert it to plaintext so you can login:

```bash
echo "cy44SDZZS1BLNE9Hb1o0bWVOdm9KTE1nZG4=" | base64 --decode
# s.8H6YKPK4OGoZ4meNvoJLMgdn
```

Login and you can start creating policies!
Next, what if we want to monitor the Vault cluster and alert on it?

### Install Prometheus Operator


To install the [Prometheus Operator](https://hub.helm.sh/charts/bitnami/prometheus-operator) first we need to add the helm repository.
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
# "bitnami" has been added to your repositories
```

Then let's create a namespace to install the chart in and switch to that namespace:
```bash
kubectl create ns prometheus
# namespace/prometheus created
kubectl config set-context --current --namespace=prometheus
# Context "kbambridge@vault.us-west-2.eksctl.io" modified.
```

Install the chart:

```bash
helm install prometheus-operator bitnami/prometheus-operator
# NAME: prometheus-operator
# LAST DEPLOYED: Sun May 24 15:01:23 2020
# NAMESPACE: prometheus
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
# NOTES: ...
```

You can look at the notes to see some helpful commands to check the status of the install.

Take a look at the pods:
```bash
kubectl get pods
# NAME                                                      READY   STATUS    RESTARTS   AGE
# alertmanager-prometheus-operator-alertmanager-0           2/2     Running   0          2m24s
# prometheus-operator-kube-state-metrics-79c9b8d664-rqpjs   1/1     Running   0          2m33s
# prometheus-operator-node-exporter-7fwdl                   1/1     Running   0          2m33s
# prometheus-operator-node-exporter-bsbh5                   1/1     Running   0          2m33s
# prometheus-operator-node-exporter-zhwq4                   1/1     Running   0          2m33s
# prometheus-operator-operator-dd58fbbd8-nz9rn              1/1     Running   0          2m33s
# prometheus-prometheus-operator-prometheus-0               3/3     Running   1          2m24s
```

There should be some new CRDs:
```bash
kubectl get crd | grep monitoring
# alertmanagers.monitoring.coreos.com     2020-05-24T22:01:18Z
# podmonitors.monitoring.coreos.com       2020-05-24T22:01:18Z
# prometheuses.monitoring.coreos.com      2020-05-24T22:01:18Z
# prometheusrules.monitoring.coreos.com   2020-05-24T22:01:18Z
# servicemonitors.monitoring.coreos.com   2020-05-24T22:01:18Z
# thanosrulers.monitoring.coreos.com      2020-05-24T22:01:19Z
```

For this tutorial we are only going to use `ServiceMonitor` and `Prometheus`.
Next install the `Prometheus` in the `vault` namespace:
```bash
kubectl config set-context --current --namespace=vault
# Context "kbambridge@vault.us-west-2.eksctl.io" modified.
kubectl apply -f ./k8s/prometheus.yaml
# configmap/vault-agent-config created
# serviceaccount/prometheus created
# prometheus.monitoring.coreos.com/prometheus created
# rolebinding.rbac.authorization.k8s.io/prometheus created
```

Note the additional `ServiceAccount` and `RoleBinding` for Prometheus in addition to the `vault-agent-config` for configuring the authentication to vault to scrape the metrics.

Now lets take a look at prometheus:
```bash
kubectl port-forward svc/prometheus-operated 9090
# Forwarding from 127.0.0.1:9090 -> 9090
# Forwarding from [::1]:9090 -> 9090
```

Lets see if the target is being hit properly: http://localhost:9090/targets
Oh wait, there aren't any targets! Lets fix that by changing the setting of  `serviceMonitorEnabled: false` to `true` in the [./k8s/vault-raft.yaml](./k8s/vault-raft.yaml) then reapply it:
```bash
kubectl apply -f ./k8s/vault-raft.yaml
# vault.vault.banzaicloud.com/vault configured
```
Now check if it was created:

```bash
kubectl get servicemonitor
# NAME    AGE
# vault   45s
```
Nice! Now check http://localhost:9090/targets again.

If you see 2/6 targets healthy, that should be correct because only one instance is the leader. This is so there aren't duplicate metrics.

### Visualizing your metrics with Grafana

Lets install [Grafana](https://hub.helm.sh/charts/bitnami/grafana) in another namespace:
```bash
kubectl create ns grafana
kubectl config set-context --current --namespace=grafana
helm install grafana bitnami/grafana
```

Get the admin credentials:
```bash
echo "Password: $(kubectl get secret grafana-admin --namespace grafana -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 --decode)"
```

Port-forward to access as usual:
```bash
kubectl port-forward svc/grafana 3000
```

Go to http://localhost:3000 and login, add the Prometheus data source we created by using the url:

```
http://prometheus-operated.vault.svc.cluster.local:9090
```

Note that this is [FQDN in Kubernetes](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

Then import a new dashboard and put in id [`7700`](https://grafana.com/grafana/dashboards/7700) for the vaults dashboard.

As you start creating and using tokens you should see these metrics change.

Thanks for exploring Vault in Kubernetes!

### Allowing External Traffic

If you want the UI to be accessible from outside the cluster we'll have to create a Service of type `LoadBalancer`. This is usually done with an [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/). A standard one is [`Nginx Ingress`](https://hub.helm.sh/charts/nginx/nginx-ingress). Install that chart into the `ingress-nginx` namespace and then create [`Ingress`](https://kubernetes.io/docs/concepts/services-networking/ingress/) objects for everything you would like to expose through that reverse proxy.


## Cleanup

```bash
eksctl delete cluster $CLUSTER_NAME
```
