# linkerd Installation

In these steps, we'll do a walk-through around how to install Linkerd within your Kubernetes cluster. 

Validate you're running a recent version of Kubernetes:

```console
[root@osc01 ~]# kubectl version --short
Client Version: v1.12.4+icp
error: You must be logged in to the server (the server has asked for the client to provide credentials)
```

NOTE: The error message from above states that you will need to enable non-expiring token for your kubernetes environment.

To do this, update `bashrc` file with the following and run the `source` command to validate the changes are in.


```console
[root@osc01 ~]# vi ~/.bashrc
...
MYKUBE=$HOME/.mykube
export KUBECONFIG=$MYKUBE/config
kubectl config use-context servicemesh

[root@osc01 ~]# source ~/.bashrc
```

If you check the version, or run any other `kubectl` the error will go away.

```bash
[root@osc01 ~]# kubectl version --short
Client Version: v1.12.4+icp
Server Version: v1.12.4+icp
```

## Install the CLI

Download the CLI that will interact with Linkerd, inclusing control plane installation within Kubernetes.

```console
[root@osc01 cluster]# curl -sL https://run.linkerd.io/install | sh
Downloading linkerd2-cli-stable-2.3.0-linux...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   622    0   622    0     0   2195      0 --:--:-- --:--:-- --:--:--  2197
100 29.8M  100 29.8M    0     0  3709k      0  0:00:08  0:00:08 --:--:-- 4225k

Download complete!, validating checksum...
Checksum valid.

Linkerd was successfully installed 🎉

Add the linkerd CLI to your path with:

  export PATH=$PATH:$HOME/.linkerd2/bin

Now run:

  linkerd check --pre                     # validate that Linkerd can be installed
  linkerd install | kubectl apply -f -    # install the control plane into the 'linkerd' namespace
  linkerd check                           # validate everything worked!
  linkerd dashboard                       # launch the dashboard

Looking for more? Visit https://linkerd.io/2/next-steps
```

Next, add `linkerd` to bashrc path

```console
export PATH=$PATH:$HOME/.linkerd2/bin
```

Validate Linkerd client version, for now, the server version will not be available because the control plane hasn't been installed yet.

```console
[root@osc01 cluster]# linkerd version
Client version: stable-2.3.0
Server version: unavailable
```

## Validate Kubernetes Cluster

Kubernetes cluster on ICP is configured in different ways. To validate Linkerd control plane can be installed correctly, run the `Linkerd check` on the kubernetes cluster.

```console
[root@osc01 cluster]# linkerd check --pre
kubernetes-api
--------------
√ can initialize the client
√ can query the Kubernetes API

kubernetes-version
------------------
√ is running the minimum Kubernetes API version
√ is running the minimum kubectl version

pre-kubernetes-setup
--------------------
√ control plane namespace does not already exist
√ can create Namespaces
√ can create ClusterRoles
√ can create ClusterRoleBindings
√ can create CustomResourceDefinitions
√ can create ServiceAccounts
√ can create Services
√ can create Deployments
√ can create ConfigMaps

pre-kubernetes-capability
-------------------------
√ has NET_ADMIN capability

linkerd-version
---------------
√ can determine the latest version
√ cli is up-to-date

Status check results are √
```

## Install Linkerd within cluster

Now that the CLi is installed and kubernetes environment checks out, install the Linkerd lightweight control plane within a dedicated namespace `linkerd`

First, create the `linkerd` namespace

```bash
[root@osc01 linkerd-scripts]# kubectl create namespace linkerd
namespace/linkerd created
```

Apply `gcr.io` and `docker.io` Image policies and `cluster role` admin policy:

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f 01-create-gcr-image-policy.yaml
clusterimagepolicy.securityenforcement.admission.cloud.ibm.com/gcr-image-policy created
[root@osc01 linkerd-scripts]# kubectl apply -f 02-create-docker-image-policy.yaml
clusterimagepolicy.securityenforcement.admission.cloud.ibm.com/docker-image-policy created
[root@osc01 linkerd-scripts]# kubectl apply -f 03-create-cluster-role-policy.yaml
clusterrolebinding.rbac.authorization.k8s.io/linkerd-cluster-role-binding created
```

Install Linkerd

```bash
[root@osc01 linkerd-scripts]# linkerd install | kubectl apply -f -
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
namespace/linkerd configured
configmap/linkerd-config created
serviceaccount/linkerd-identity created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-identity unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-identity unchanged
service/linkerd-identity created
secret/linkerd-identity-issuer created
deployment.extensions/linkerd-identity created
serviceaccount/linkerd-controller created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-controller unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-controller unchanged
service/linkerd-controller-api created
service/linkerd-destination created
deployment.extensions/linkerd-controller created
customresourcedefinition.apiextensions.k8s.io/serviceprofiles.linkerd.io unchanged
serviceaccount/linkerd-web created
service/linkerd-web created
deployment.extensions/linkerd-web created
serviceaccount/linkerd-prometheus created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-prometheus unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-prometheus unchanged
service/linkerd-prometheus created
deployment.extensions/linkerd-prometheus created
configmap/linkerd-prometheus-config created
serviceaccount/linkerd-grafana created
service/linkerd-grafana created
deployment.extensions/linkerd-grafana created
configmap/linkerd-grafana-config created
serviceaccount/linkerd-sp-validator created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-sp-validator unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-sp-validator configured
service/linkerd-sp-validator created
deployment.extensions/linkerd-sp-validator created
[root@osc01 linkerd-scripts]#
```

Run the Linkerd check to validate its running:

```bash
[root@osc01 ~]# linkerd check
kubernetes-api
--------------
√ can initialize the client
√ can query the Kubernetes API

kubernetes-version
------------------
√ is running the minimum Kubernetes API version
√ is running the minimum kubectl version

linkerd-existence
-----------------
√ control plane namespace exists
√ controller pod is running
√ can initialize the client
√ can query the control plane API

linkerd-api
-----------
√ control plane pods are ready
√ control plane self-check
√ [kubernetes] control plane can talk to Kubernetes
√ [prometheus] control plane can talk to Prometheus
√ no invalid service profiles

linkerd-version
---------------
√ can determine the latest version
√ cli is up-to-date

control-plane-version
---------------------
√ control plane is up-to-date
√ control plane and cli versions match

Status check results are √
```

Validate `Linkerd` installed components:

```bash
[root@osc01 ~]# kubectl -n linkerd get deploy
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
linkerd-controller     1         1         1            1           28m
linkerd-grafana        1         1         1            1           27m
linkerd-identity       1         1         1            1           28m
linkerd-prometheus     1         1         1            1           27m
linkerd-sp-validator   1         1         1            1           27m
linkerd-web            1         1         1            1           27m
```

Validate `Linkerd` running pods:

```bash
[root@osc01 ~]# kubectl -n linkerd get pods
NAME                                    READY   STATUS    RESTARTS   AGE
linkerd-controller-67964c6755-ppjjm     4/4     Running   0          6m29s
linkerd-grafana-67475f7754-q2822        2/2     Running   0          6m29s
linkerd-identity-7546bd4bd7-j8krt       2/2     Running   0          6m29s
linkerd-prometheus-6d4c5fb78-zdv9x      2/2     Running   0          6m29s
linkerd-sp-validator-57d8bb9c5b-2gn4h   2/2     Running   0          6m29s
linkerd-web-596b84c7d5-mskhd            2/2     Running   0          6m29s
```

## Explore Linkerd

Now that the control plane is installed, go to `Linkerd dashboard` by running the following command and navigate to the provided URL within the Virtual Machine.

In this case, the URL is: http://127.0.0.1:50750/

```bash
[root@osc01 linkerd-scripts]# linkerd dashboard &
[3] 32469
[root@osc01 linkerd-scripts]# Linkerd dashboard available at:
http://127.0.0.1:36173
Grafana dashboard available at:
http://127.0.0.1:36173/grafana
Opening Linkerd dashboard in the default browser
START /usr/bin/firefox "http://127.0.0.1:36173"
Failed to open connection to "session" message bus: Unable to autolaunch a dbus-daemon without a $DISPLAY for X11
Running without a11y support!
Error: no DISPLAY environment variable specified
xdg-open: no method available for opening 'http://127.0.0.1:36173'
Failed to open Linkerd dashboard automatically
Visit http://127.0.0.1:36173 in your browser to view the dashboard
```

## Install EMOJI Demo App

Install the following demo application to get an overview of Linkerd functionality. 

Create a namespace

```bash
[root@osc01 cluster]# kubectl create ns emojivoto
namespace/emojivoto created
```

Create cluster binding role for `emojivoto` service.

```bash
[root@osc01 cluster]# kubectl apply -f 04-create-podsecuritypolicy-emoji.yaml -n emojivoto
podsecuritypolicy.policy/emoji-example configured
```

Download the `emojivoto` YAML 

```bash
[root@osc01 cluster]# curl -sL https://run.linkerd.io/emojivoto.yml > emoji.yaml
```

Inject the `emojivoto` service with Linkerd proxy for installing the demo application. The `inject` command augments resources to include data plane proxy. Similar to install, inject executes a rolling deploy for every pod with sidecar proxies without any downtime.

```bash
[root@osc01 cluster]# linkerd inject emoji.yaml | kubectl apply -f -
namespace "emojivoto" skipped
serviceaccount "emoji" skipped
serviceaccount "voting" skipped
serviceaccount "web" skipped
deployment "emoji" injected
service "emoji-svc" skipped
deployment "voting" injected
service "voting-svc" skipped
deployment "web" injected
service "web-svc" skipped
deployment "vote-bot" injected
namespace/emojivoto unchanged
serviceaccount/emoji unchanged
serviceaccount/voting unchanged
serviceaccount/web unchanged
deployment.apps/emoji configured
service/emoji-svc unchanged
deployment.apps/voting configured
service/voting-svc unchanged
deployment.apps/web configured
service/web-svc unchanged
deployment.apps/vote-bot configured
```

Validate the pods are created for `emoji` app

```bash
[root@osc01 linkerd-scripts]# kgp -n emojivoto
NAME                        READY   STATUS    RESTARTS   AGE
emoji-68b46d6ff6-fhk6h      2/2     Running   0          12m
vote-bot-76566bd794-7rtg5   2/2     Running   0          12m
voting-6f575574bb-zx4sk     2/2     Running   0          12m
web-65d79dd6b-gj4zr         2/2     Running   0          12m
```

Validate the replica sets for `emojivoto`

```bash
[root@osc01 linkerd-scripts]# kgrs -n emojivoto
NAME                  DESIRED   CURRENT   READY   AGE
emoji-68b46d6ff6      1         1         1       30m
vote-bot-76566bd794   1         1         1       30m
voting-6f575574bb     1         1         1       30m
web-65d79dd6b         1         1         1       30m
```

Enable port forwarding to access the emjoi app through local port 8080

!DIDN'T WORK
```console
[root@osc01 ~]# kubectl -n emojivoto port-forward web-65d79dd6b-gj4zr 8080:80
```

Finally, validate everything works within `emojivota` data plane through the following command

```bash
[root@osc01 ~]# linkerd -n emojivoto check --proxy
kubernetes-api
--------------
√ can initialize the client
√ can query the Kubernetes API

kubernetes-version
------------------
√ is running the minimum Kubernetes API version
√ is running the minimum kubectl version

linkerd-existence
-----------------
√ control plane namespace exists
√ controller pod is running
√ can initialize the client
√ can query the control plane API

linkerd-api
-----------
√ control plane pods are ready
√ control plane self-check
√ [kubernetes] control plane can talk to Kubernetes
√ [prometheus] control plane can talk to Prometheus
√ no invalid service profiles

linkerd-version
---------------
√ can determine the latest version
√ cli is up-to-date

linkerd-data-plane
------------------
√ data plane namespace exists
√ data plane proxies are ready
√ data plane proxy metrics are present in Prometheus
√ data plane is up-to-date
√ data plane and cli versions match

Status check results are √
```

Next, you can see the `emojivoto` namespace stats within Linkerd dashboard. Here are some high level stats about the service:

It shows success & request rates, and latency distribution. 

```bash
[root@osc01 ~]# linkerd -n emojivoto stat deploy
NAME       MESHED   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99   TCP_CONN
emoji         1/1   100.00%   2.0rps           1ms           1ms           1ms          2
vote-bot      1/1         -        -             -             -             -          -
voting        1/1    83.05%   1.0rps           1ms           1ms           1ms          2
web           1/1    89.17%   2.0rps           5ms           9ms          10ms          2
```

Running the `top` command provides real-time `per-path` basis stats on above metrics. It will look something like this. 

```console
(press q to quit)

Source                     Destination              Method      Path
web-65d79dd6b-gj4zr        emoji-68b46d6ff6-fhk6h   POST        /emojivoto.v1.EmojiService/ListAll
vote-bot-76566bd794-7rtg5  web-65d79dd6b-gj4zr      GET         /api/list
web-65d79dd6b-gj4zr        emoji-68b46d6ff6-fhk6h   POST        /emojivoto.v1.EmojiService/FindByShortcode
vote-bot-76566bd794-7rtg5  web-65d79dd6b-gj4zr      GET         /api/vote
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteDoughnut
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteWave
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteDog
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteGolfingMan
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteRaisedHands
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteMassageWoma
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteSanta
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VotePizza
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteTurkey
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteSteamLocomo
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteFlightDepar
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteSnail
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VotePig
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteConstructio
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteMoneyMouthF
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/Vote100
web-65d79dd6b-gj4zr        voting-6f575574bb-zx4sk  POST        /emojivoto.v1.VotingService/VoteTaco
```

To dig deeper, enter `tap` to show request streams across a single pod within the `emojivoto` namespace.

It will look something like this

```
end id=15:21 proxy=in  src=10.1.230.250:58966 dst=10.1.230.237:80 tls=true duration=553µs response-length=4513B
req id=15:23 proxy=in  src=10.1.230.250:58966 dst=10.1.230.237:80 tls=true :method=GET :authority=web-svc.emojivoto:80 :path=/api/vote
req id=15:24 proxy=out src=10.1.230.237:38136 dst=10.1.230.252:8080 tls=true :method=POST :authority=emoji-svc.emojivoto:8080 :path=/emojivoto.v1.EmojiService/FindByShortcode
rsp id=15:24 proxy=out src=10.1.230.237:38136 dst=10.1.230.252:8080 tls=true :status=200 latency=2807µs
end id=15:24 proxy=out src=10.1.230.237:38136 dst=10.1.230.252:8080 tls=true grpc-status=OK duration=77µs response-length=30B
req id=15:25 proxy=out src=10.1.230.237:48742 dst=10.1.230.235:8080 tls=true :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VoteManInTuxedo
rsp id=15:25 proxy=out src=10.1.230.237:48742 dst=10.1.230.235:8080 tls=true :status=200 latency=2690µs
end id=15:25 proxy=out src=10.1.230.237:48742 dst=10.1.230.235:8080 tls=true grpc-status=OK duration=113µs response-length=5B
rsp id=15:23 proxy=in  src=10.1.230.250:58966 dst=10.1.230.237:80 tls=true :status=200 latency=8506µs
end id=15:23 proxy=in  src=10.1.230.250:58966 dst=10.1.230.237:80 tls=true duration=69µs response-length=0B
req id=15:26 proxy=in  src=10.1.230.250:58966 dst=10.1.230.237:80 tls=true :method=GET :authority=web-svc.emojivoto:80 :path=/api/list
req id=15:27 proxy=out src=10.1.230.237:38136 dst=10.1.230.252:8080 tls=true :method=POST :authority=emoji-svc.emojivoto:8080 :path=/emojivoto.v1.EmojiService/ListAll
rsp id=15:27 proxy=out src=10.1.230.237:38136 dst=10.1.230.252:8080 tls=true :status=200 latency=1141µs
end id=15:27 proxy=out src=10.1.230.237:38136 dst=10.1.230.252:8080 tls=true grpc-status=OK duration=263µs response-length=2140B
rsp id=15:26 proxy=in  src=10.1.230.250:58966 dst
...
```

Note: The above commands for `stat`, `top` and `tap` are all available for view within the Linkerd dashboard as well.

To visualize metrics that are not real-time, Grafana is used for metrics collected by Prometheus.

# Adding the Bookapp service

Deploy the Bookapp microservice and inject it with Linkerd proxy sidecar.

If the booksapp is already deploy, validate its pods, deployment and replica sets:

```bash
[root@osc01 cluster]# kgp -n linkerd-lab
NAME                       READY   STATUS    RESTARTS   AGE
authors-74695fc659-4gxtm   1/1     Running   3          21d
books-6d77d575bc-khgkx     1/1     Running   3          21d
traffic-76b49b884d-jprbc   1/1     Running   3          21d
webapp-d99b48fc5-9sfvh     1/1     Running   3          21d
webapp-d99b48fc5-gzptb     1/1     Running   3          21d
webapp-d99b48fc5-vhjpg     1/1     Running   3          21d
[root@osc01 cluster]# kgsvc -n linkerd-lab
NAME      TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
authors   ClusterIP      None         <none>        7001/TCP         21d
books     ClusterIP      None         <none>        7002/TCP         21d
webapp    LoadBalancer   10.0.0.19    <pending>     7000:30789/TCP   21d
[root@osc01 cluster]# kgrs -n linkerd-lab
NAME                 DESIRED   CURRENT   READY   AGE
authors-74695fc659   1         1         1       21d
books-6d77d575bc     1         1         1       21d
traffic-76b49b884d   1         1         1       21d
webapp-d99b48fc5     3         3         3       21d
```

Inject the Linkerd proxy into Bookapp microservice:

```bash
[root@osc01 cluster]# linkerd inject booksapp.yaml | kubectl apply -f -

service "webapp" skipped
deployment "webapp" injected
service "authors" skipped
deployment "authors" injected
service "books" skipped
deployment "books" injected
deployment "traffic" injected

service/webapp created
deployment.extensions/webapp created
service/authors created
deployment.extensions/authors created
service/books created
deployment.extensions/books created
deployment.extensions/traffic created
```

[Will circle back]

# Automating Linkerd Injection

Automatic proxy injection is implemented with admission webhook by updating resources within a namespace using `kubectl apply` or `kubectl edit`.

Since injection happens at admission and on specific resource, kubernetes will not call mutating webhook until updates are available for every resource.

To leverage automatic injection, validate the `admissionregistration` 

```bash
[root@osc01 cluster]# kubectl api-versions | grep admissionregistration
admissionregistration.k8s.io/v1alpha1
admissionregistration.k8s.io/v1beta1
```

By default, automatic proxy injection is disabled within installing Linkerd control plane. Enable it by using the flag `--proxy-auto-inject` flag.

Use the following if Linkerd hasn't been installed:

`linkerd install --proxy-auto-inject | kubectl apply -f -`

Use the following if Linkerd is already installed, but need to upgrade with automatic proxy injection

```bash
[root@osc01 cluster]# linkerd upgrade --proxy-auto-inject | kubectl apply -f -

√ You're on your way to upgrading Linkerd!
Visit this URL for further instructions: https://linkerd.io/upgrade/#nextsteps

namespace/linkerd configured
configmap/linkerd-config configured
serviceaccount/linkerd-identity unchanged
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-identity unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-identity unchanged
service/linkerd-identity unchanged
secret/linkerd-identity-issuer unchanged
deployment.extensions/linkerd-identity configured
serviceaccount/linkerd-controller unchanged
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-controller unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-controller unchanged
service/linkerd-controller-api unchanged
service/linkerd-destination unchanged
deployment.extensions/linkerd-controller configured
customresourcedefinition.apiextensions.k8s.io/serviceprofiles.linkerd.io unchanged
serviceaccount/linkerd-web unchanged
service/linkerd-web unchanged
deployment.extensions/linkerd-web configured
serviceaccount/linkerd-prometheus unchanged
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-prometheus unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-prometheus unchanged
service/linkerd-prometheus unchanged
deployment.extensions/linkerd-prometheus configured
configmap/linkerd-prometheus-config unchanged
serviceaccount/linkerd-grafana unchanged
service/linkerd-grafana unchanged
deployment.extensions/linkerd-grafana configured
configmap/linkerd-grafana-config unchanged
deployment.apps/linkerd-proxy-injector created
serviceaccount/linkerd-proxy-injector created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-proxy-injector created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-proxy-injector created
service/linkerd-proxy-injector created
serviceaccount/linkerd-sp-validator unchanged
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-sp-validator unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-sp-validator configured
service/linkerd-sp-validator unchanged
deployment.extensions/linkerd-sp-validator configured
```

A new `injector` pod has been created and verify it is working correctly

```bash
[root@osc01 cluster]# kubectl -n linkerd get deploy/linkerd-proxy-injector svc/linkerd-proxy-injector
NAME                                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/linkerd-proxy-injector   1         1         1            1           112s

NAME                             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/linkerd-proxy-injector   ClusterIP   10.0.0.158   <none>        443/TCP   111s
```

```bash
[root@osc01 cluster]# kgp -n linkerd
NAME                                      READY   STATUS    RESTARTS   AGE
linkerd-controller-56546b4bfd-jqr55       4/4     Running   10         6h17m
linkerd-grafana-77cdcd8879-dpkxx          2/2     Running   2          6h17m
linkerd-identity-7cd6f7fcd6-nccsl         2/2     Running   2          6h17m
linkerd-prometheus-84c68f7c67-f2xrb       2/2     Running   2          6h17m
linkerd-proxy-injector-766f8cc66b-dpsdm   2/2     Running   0          60s
linkerd-sp-validator-66644b9dbf-xvkxs     2/2     Running   2          6h17m
linkerd-web-68d9fbd5d4-9tfgn              2/2     Running   2          6h17m
```

## Configuration

Overall, automatic proxy injection can only be performed on pods if the `linkerd.io/inject: enabled` annoation is applied. If a namespace has been configured with auto-injection, it can be possible to disable injection by applying the `linkerd.io/inject: disabled`.

For instance, let's create a sample namespace and deploy the sample application YAML with automatic proxy injection.

```bash
[root@osc01 linkerd-scripts]# kubectl create ns sample-inject-enabled-ns
namespace/sample-inject-enabled-ns created

[root@osc01 linkerd-scripts]# kubectl apply -f 05-apply-proxy-inject-sample-ns.yaml -n sample-inject-enabled-ns
namespace/sample-inject-enabled-ns configured
```

Create a new deployment within the sample namespace.

```bash
[root@osc01 linkerd-scripts]# kubectl -n sample-inject-enabled-ns run helloworld --image=buoyantio/helloworld
deployment.apps/helloworld created
```

Deploy the pod security policy for sample application.

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f 06-create-podsecuritypolicy-sample.yaml -n sample-inject-enabled-ns
podsecuritypolicy.policy/sample-example configured
```

Deploy the cluster role for sample application

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f 07-create-cluster-role-policy-sample.yaml
clusterrolebinding.rbac.authorization.k8s.io/sample-cluster-role-binding created
```

Validate the sample application running pods, replica sets and deployment pod includes the `linkerd-proxy` container.

```bash
[root@osc01 linkerd-scripts]# kgp -n sample-inject-enabled-ns
NAME                        READY   STATUS    RESTARTS   AGE
helloworld-f5b7bfc5-fbbzn   2/2     Running   0          3m29s
```

```bash
[root@osc01 linkerd-scripts]# kgrs -n sample-inject-enabled-ns
NAME                  DESIRED   CURRENT   READY   AGE
helloworld-f5b7bfc5   1         1         1       31m
```

```bash
[root@osc01 linkerd-scripts]# kubectl -n sample-inject-enabled-ns get po -l run=helloworld -o jsonpath='{.items[0].spec.containers[*].name}'
helloworld linkerd-proxy
```

It is also possible to explicitly configure auto-injection for all pods in a deployment. Even if that specific deployment is running in a namespace that hasn't been configured with auto-inject. 

To do this, create a new deployment in the existing `default` namespace and add the `linkerd.io/inject: enabled` to the deployment's pod spec.

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f 08-apply-proxy-inject-default-ns.yaml -n default
deployment.extensions/helloworld-enabled created
```

Once the configuraiton is applied, the deployment pod is injected with a `linkerd-proxy` container. To verify, run the following and confirm its running.

```bash
[root@osc01 linkerd-scripts]# kubectl get po -l run=helloworld-enabled -o jsonpath='{.items[0].spec.containers[*].name}'
helloworld-enabled linkerd-proxy
```

Its possible to disable auto-injection for all pods in a deployment. Even if that deployment is running in a namespace that is configured for auto-inject. To do this, add the `linkerd.io/inject: disabled` annotation to the deployment's pod sec.

Create this deployment in the `sample-inject-enabled-ns` ns.

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f 09-disable-proxy-inject-sample-ns.yaml -n sample-inject-enabled-ns
deployment.extensions/helloworld-disabled created
```

Validate the deployment is running. Note: this doesn't change or affect the previous deployment of enabling the proxy injection.

```bash
[root@osc01 linkerd-scripts]# kgp -n sample-inject-enabled-ns
NAME                                   READY   STATUS    RESTARTS   AGE
helloworld-disabled-85d7bfff9c-ngmct   1/1     Running   0          64s
helloworld-f5b7bfc5-fbbzn              2/2     Running   0          17m
```

Once the configuration is applied, verify the deployment pod is NOT injected with a `linkerd-proxy` container:

```bash
[root@osc01 linkerd-scripts]# kubectl -n sample-inject-enabled-ns get po -l run=helloworld-disabled -o jsonpath='{.items[0].spec.containers[*].name}'
helloworld-disabled
```

# Configuring Retries

For Linkerd to do automatic retries of failures, additional information is required for which requests will need to be re-tried and how many times should the re-try happen.

Unfortunately, automatic re-tries for a request that changes state could impact user experience and increase load to the system. Many request retries could potentially take down the system because of the re-tries not providing enough time for recovery.

Admins can enable a retry budget, that limits # of re-tries that can be performed against a service as percentage of original requests.

This prevents retires from overwhelming your system. By default, re-tries may add additional 20% of request load.

Settings can be adjusted by applying a `retrybudget` on service profile.

```yaml
spec:
  retryBudget:
    retryRatio: 0.2
    minRetriesPerSecond: 10
    ttl: 10s
```

# Configuring Timeouts

Linkerd wait time can be limited through timeouts before failing an outgoing requests to another service.

Every route may define a timeout that specifies a max amount of time to wait for a response (including re-tries) to complete after the request is sent. Once a timeout is reached, Linkerd will cancel the request and return a 504 response. Default timeout is 10 seconds.

```yaml
spec:
  routes:
  - condition:
      method: HEAD
      pathRegex: /authors/[^/]*\.json
    name: HEAD /authors/{id}.json
    timeout: 300ms
```

# Debugging EMOJI service

Install Linkerd and the `emoji` demo app within a kubernetes cluster. The setup can be seen on the linkerd dashboard with a dedicated `emojivoto` namespace, including deployments. 

Navigate to deployment page for web, here you see the web deployment is taking traffic from `vote-bot`, deployment included with `emojivoto` to continually generate low level of live traffic. Web deployment also has two outgoing dependency, `emjoi` and `voting`.

There are two calls that shown in the web

1) `vote-bot` calls to the `/api/vote`
2) `VoteDoughnut` calls from the web deployment to its dependent deployment, `voting`. 

Since `/api/vote` is an  incoming call and `VoteDoughnut` is outgoing, the voting deployment is failing requests. A failure in dependent deployment is causing the errors that web is returning.

Click on the `tap` icon, these are the live requests that match only this endpoint. There will be an unknown status under the `GRPC status` column. This is a common error responses within Linkerd and they are aware of gRPC's response classification without any configuration.

# Deploy BooksApp Service

The Booksapp service is a ruby application that manages a `bookshelf`. It has many microservices that uses JSON over HTTP to communicate with other services.

The three services are:

1) `webapp`, which is the frontend microservice
2) `authors`, an API to manage authors in the system
3) `books`, an API to manage books in the system

## Install the microservice

Install booksapp by creating namespace, dedicated pod security policy and injecting linkerd proxy:

```bash
[root@osc01 linkerd-scripts]# kubectl create ns booksapp
namespace/booksapp created
```

Apply the `booksapp` namespace pod security policy

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f 10-create-podsecuritypolicy-booksapp.yaml -n booksapp
podsecuritypolicy.policy/booksapp-example created
```

Apply the cluster role policy for service account `booksapp`

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f 11-create-cluster-role-policy-booksapp.yaml
clusterrolebinding.rbac.authorization.k8s.io/booksapp-cluster-role-binding created
```

Inject the `booksapp` service with automatic proxy and install the microservice.

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f booksapp.yaml -n booksapp
service/webapp created
deployment.extensions/webapp created
service/authors created
deployment.extensions/authors created
service/books created
deployment.extensions/books created
deployment.extensions/traffic created
```

Validate the booksapp services, pods and replica sets are running:

```bash
[root@osc01 linkerd-scripts]# kubectl -n booksapp get all
NAME                           READY   STATUS    RESTARTS   AGE
pod/authors-74695fc659-2ftlq   1/1     Running   0          14s
pod/books-6d77d575bc-pnprn     1/1     Running   0          14s
pod/traffic-76b49b884d-mhclq   1/1     Running   0          13s
pod/webapp-d99b48fc5-4p64q     1/1     Running   0          14s
pod/webapp-d99b48fc5-5vxb2     1/1     Running   0          14s
pod/webapp-d99b48fc5-h2rnh     1/1     Running   0          14s

NAME              TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
service/authors   ClusterIP      None         <none>        7001/TCP         14s
service/books     ClusterIP      None         <none>        7002/TCP         14s
service/webapp    LoadBalancer   10.0.0.76    <pending>     7000:31245/TCP   14s

NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/authors   1         1         1            1           14s
deployment.apps/books     1         1         1            1           14s
deployment.apps/traffic   1         1         1            1           14s
deployment.apps/webapp    3         3         3            3           14s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/authors-74695fc659   1         1         1       14s
replicaset.apps/books-6d77d575bc     1         1         1       14s
replicaset.apps/traffic-76b49b884d   1         1         1       13s
replicaset.apps/webapp-d99b48fc5     3         3         3       14s
```

Validate linkerd `webapp` services are ready for traffic

```bash
[root@osc01 linkerd-scripts]# kubectl -n booksapp rollout status deploy webapp
deployment "webapp" successfully rolled out
```

Now that the `booksapp` service is installed, enable port forward to access the app from your browser within virtual machine.

```console
[root@osc01 linkerd-scripts]# kubectl -n booksapp port-forward svc/webapp 7000 &
[1] 29362
[root@osc01 linkerd-scripts]# Forwarding from 127.0.0.1:7000 -> 7000
Handling connection for 7000
Handling connection for 7000
...
```

Within the booksapp, if you try to add a book, it will fail 50% of the time and throw out an `Internal Service Error`. This is intentional, and idea is to show that since the application is running fine, there are returning errors.

## Add Linkerd

Apply `linkerd inject` adds:

1) an `initContainer` that sets up iptables to allow forwarding for all incoming and outgoing traffic through Linkerd's sidecar proxy.
2) a `container` that runs the proxy.

```bash
[root@osc01 ~]# kubectl get -n booksapp deploy -o yaml | linkerd inject - | kubectl apply -f -

deployment "authors" injected
deployment "books" injected
deployment "traffic" injected
deployment "webapp" injected

deployment.extensions/authors configured
deployment.extensions/books configured
deployment.extensions/traffic configured
deployment.extensions/webapp configured
```

Validate the pod count has incremented from 1 to 2.

```bash
NAME                       READY   STATUS    RESTARTS   AGE
authors-86d54dd46b-jwcqc   2/2     Running   0          82s
books-84dfdfb49b-4zq2q     2/2     Running   0          80s
traffic-58b889c65b-2fdrb   2/2     Running   0          79s
webapp-647fff74b7-2ckz4    2/2     Running   0          66s
webapp-647fff74b7-6hvjw    2/2     Running   0          77s
webapp-647fff74b7-95kvj    2/2     Running   0          77s
```

Now that the linkerd proxy is inject, lets debug the `booksapp` service to identify the app failures.

## Debugging

Navigate to the linkerd dashboard and click on `booksapp` namespace to show its designated services.

Click on `webapp` service, which takes you to the deployment page.

Here you can see the service topology of which booksapp services is the webapp communicating with.

The great feature about this view is that it provides live calls for all service to service requests and responses and the necessary success rate metrics.

## Service Profiles

If you're facing intermittent issues, service profiles can be leveraged to provide additional troubleshooting about your servies.

These profiles define routes and allow collection of metrics / route basis. Prometheus is a great platform to store such metrics.

To setup a service profile, use existing OpenAPI (Swagger) specs.

In the example below, create a service profile for `webapp`:

```bash
[root@osc01 linkerd-scripts]# curl -sL https://run.linkerd.io/booksapp/webapp.swagger | linkerd -n booksapp profile --open-api - webapp | kubectl -n booksapp apply -f -
serviceprofile.linkerd.io/webapp.booksapp.svc.cluster.local created
```

The command does three things:

1) Fetches the swagger specification for `webapp`
2) Coverts the spec to service profile using `profile` command
3) Apply the configuration to the cluster

Validate the serviceprofile was created for `webapp`


```bash
[root@osc01 linkerd-scripts]# kubectl get serviceprofile -n booksapp
NAME                                AGE
webapp.booksapp.svc.cluster.local   5m
```

Check the serviceprofile YAML, Linkerd uses the `Host` header of requests to associate service profiles with requests. The host and it's header in this case is `webapp.booksapp.svc.cluster.local`, it will use this to lookup profile configuration.

```console
API Version:  linkerd.io/v1alpha1
Kind:         ServiceProfile
Metadata:
  Creation Timestamp:  2019-04-30T14:57:01Z
  Generation:          1
  Resource Version:    72916
  Self Link:           /apis/linkerd.io/v1alpha1/namespaces/booksapp/serviceprofiles/webapp.booksapp.svc.cluster.local
  UID:                 35d21a9e-6b58-11e9-9c20-00505632f6a0
Spec:
  Routes:
    Condition:
      Method:      GET
      Path Regex:  /
    Name:          GET /
    Condition:
      Method:      POST
      Path Regex:  /authors
    Name:          POST /authors
    Condition:
      Method:      GET
      Path Regex:  /authors/[^/]*
    Name:          GET /authors/{id}
    Condition:
      Method:      POST
      Path Regex:  /authors/[^/]*/delete
    Name:          POST /authors/{id}/delete
    Condition:
      Method:      POST
      Path Regex:  /authors/[^/]*/edit
    Name:          POST /authors/{id}/edit
    Condition:
      Method:      POST
      Path Regex:  /books
    Name:          POST /books
    Condition:
      Method:      GET
      Path Regex:  /books/[^/]*
    Name:          GET /books/{id}
    Condition:
      Method:      POST
      Path Regex:  /books/[^/]*/delete
    Name:          POST /books/{id}/delete
    Condition:
      Method:      POST
      Path Regex:  /books/[^/]*/edit
    Name:          POST /books/{id}/edit
```

Route are defined by simple conditions that contain method (`GET` for instance) with a regex to match the path. This allows the grouping of REST style resources together.

Next, create service profiles for `authors` 

```bash
[root@osc01 linkerd-scripts]# curl -sL https://run.linkerd.io/booksapp/authors.swagger | linkerd -n booksapp profile --open-api - authors | kubectl -n booksapp apply -f -
serviceprofile.linkerd.io/authors.booksapp.svc.cluster.local created
```

And service profile for `books`

```bash
[root@osc01 linkerd-scripts]# curl -sL https://run.linkerd.io/booksapp/books.swagger | linkerd -n booksapp profile --open-api - books | kubectl -n booksapp apply -f -
serviceprofile.linkerd.io/books.booksapp.svc.cluster.local created
```

Verify that all the service profiles are available through CLI or through Linkerd dashboard under `linkerd tap`. Every live request will show up with an `:authority` or `host` header.

From the CLI run the following command to see the live requests:

```bash
[root@osc01 linkerd-scripts]# linkerd -n booksapp tap deploy/webapp -o wide | grep req
req id=8:1 proxy=in  src=10.1.230.241:54042 dst=10.1.230.199:7000 tls=true :method=GET :authority=webapp:7000 :path=/authors/1353 src_res=deploy/traffic src_ns=booksapp dst_res=deploy/webapp dst_ns=booksapp rt_route=GET /authors/{id}

req id=8:2 proxy=out src=10.1.230.199:54584 dst=10.1.230.229:7001 tls=true :method=GET :authority=authors:7001 :path=/authors/1353.json src_res=deploy/webapp src_ns=booksapp dst_res=deploy/authors dst_ns=booksapp rt_route=GET /authors/{id}.json

req id=8:3 proxy=out src=10.1.230.199:39244 dst=10.1.230.245:7002 tls=true :method=GET :authority=books:7002 :path=/books.json src_res=deploy/webapp src_ns=booksapp dst_res=deploy/books dst_ns=booksapp rt_route=GET /books.json

...

```

What you see in the output are:

1) `:authority` which is the host 
2) `:path` that is being used
3) `:rt_route` is the route name.

These metrics are part of the `linkerd routes` instead of `linkerd stat`.

Accumulated metrics by Linkerd

```bash
[root@osc01 ~]# linkerd -n booksapp routes svc/webapp
ROUTE                       SERVICE   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
GET /                        webapp   100.00%   0.6rps          19ms          29ms          30ms
GET /authors/{id}            webapp   100.00%   0.6rps          16ms          27ms          29ms
GET /books/{id}              webapp   100.00%   1.2rps          18ms          36ms          39ms
POST /authors                webapp   100.00%   0.6rps          16ms          36ms          39ms
POST /authors/{id}/delete    webapp   100.00%   0.6rps          23ms          29ms          30ms
POST /authors/{id}/edit      webapp     0.00%   0.0rps           0ms           0ms           0ms
POST /books                  webapp    50.35%   2.4rps          18ms          39ms          48ms
POST /books/{id}/delete      webapp   100.00%   0.6rps          10ms          27ms          29ms
POST /books/{id}/edit        webapp    50.70%   1.2rps          75ms          98ms         100ms
[DEFAULT]                    webapp     0.00%   0.0rps           0ms           0ms           0ms
```

Service profiles can be used to observer incoming and outgoing requests, to see that:

```bash
[root@osc01 ~]# linkerd -n booksapp routes deploy/webapp --to svc/books
ROUTE                     SERVICE   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
DELETE /books/{id}.json     books   100.00%   0.6rps          13ms          19ms          20ms
GET /books.json             books   100.00%   1.1rps           8ms          19ms          20ms
GET /books/{id}.json        books   100.00%   2.1rps           8ms          20ms          28ms
POST /books.json            books    49.62%   2.2rps          22ms          39ms          40ms
PUT /books/{id}.json        books    49.21%   1.1rps          61ms          96ms          99ms
[DEFAULT]                   books     0.00%   0.0rps           0ms           0ms           0ms
```

This view shows all requests and routes that originates within the `webapp` deployment and are destined for the `books` service. 

As you can see, the  requests for books has a lower success percentage versus the other services like `author`. This shows the root causes of the internal error we were facing earlier in the demo about about not being able to add additional books. 

## Retries

Code updates or new version upgrades, can notify linkerd to retry requests for a failing endpoint. This process will increase request latencies, becuase the requests will be retried multiple times. 

Within the `booksapp` service, the success rate from `books` to `authors` is poor. To see these metrics:

```bash
[root@osc01 ~]# linkerd -n booksapp routes deploy/books --to svc/authors
ROUTE                       SERVICE   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
DELETE /authors/{id}.json   authors     0.00%   0.0rps           0ms           0ms           0ms
GET /authors.json           authors     0.00%   0.0rps           0ms           0ms           0ms
GET /authors/{id}.json      authors     0.00%   0.0rps           0ms           0ms           0ms
HEAD /authors/{id}.json     authors    47.52%   3.4rps           4ms          11ms          18ms
POST /authors.json          authors     0.00%   0.0rps           0ms           0ms           0ms
[DEFAULT]                   authors     0.00%   0.0rps           0ms           0ms           0ms
```

As noticed, the requests from books to authors has a 47% success rate and causing failures the remaining 53% of the time. 

To fix this issue, edit the `authors` service profile and make those requests re-try if a failure occurs by adding `isRetryable: true` to a specific route under specifications.

Run the following command to edit the service profile for `books`: `kubectl -n booksapp edit sp/authors.booksapp.svc.cluster.local`

```yaml
  - condition:
      method: HEAD
      pathRegex: /authors/[^/]*\.json
    name: HEAD /authors/{id}.json
    isRetryable: true ### Add this line ###
```

Save and exit the service profile. Next, Linkerd will begin to retry the requests to this route automatically. 

If you re-run the `linkerd -n booksapp routes deploy/books --to svc/authors` command, you will see the success rate percentage increase, eventually to 100% success after the re-tries are enabled.

```console
[root@osc01 ~]# linkerd -n booksapp routes deploy/books --to svc/authors
ROUTE                       SERVICE   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
DELETE /authors/{id}.json   authors     0.00%   0.0rps           0ms           0ms           0ms
GET /authors.json           authors     0.00%   0.0rps           0ms           0ms           0ms
GET /authors/{id}.json      authors     0.00%   0.0rps           0ms           0ms           0ms
HEAD /authors/{id}.json     authors   100.00%   2.2rps          12ms          26ms          29ms
POST /authors.json          authors     0.00%   0.0rps           0ms           0ms           0ms
[DEFAULT]                   authors     0.00%   0.0rps           0ms           0ms           0ms
```

If you add the `-o wide` extension, you will see a success rate of `EFFECTIVE_SUCCESS` and `ACTUAL_SUCCESS`.

```bash
[root@osc01 ~]# linkerd -n booksapp routes deploy/books --to svc/authors -o wide
ROUTE                       SERVICE   EFFECTIVE_SUCCESS   EFFECTIVE_RPS   ACTUAL_SUCCESS   ACTUAL_RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
DELETE /authors/{id}.json   authors               0.00%          0.0rps            0.00%       0.0rps           0ms           0ms           0ms
GET /authors.json           authors               0.00%          0.0rps            0.00%       0.0rps           0ms           0ms           0ms
GET /authors/{id}.json      authors               0.00%          0.0rps            0.00%       0.0rps           0ms           0ms           0ms
HEAD /authors/{id}.json     authors             100.00%          2.2rps           45.14%       4.8rps           9ms          27ms          29ms
POST /authors.json          authors               0.00%          0.0rps            0.00%       0.0rps           0ms           0ms           0ms
[DEFAULT]                   authors               0.00%          0.0rps            0.00%       0.0rps           0ms           0ms           0ms
```

The difference between the two success rates is, how well are the retries working. 
`EFFECTIVE_RPS` and `ACTUAL_RPS` show many requests are being sent to this destination service and how many are receive by the Linkerd proxy.

If you notice, enabling retries, the latency of `P95` and `P99` has also increased. This behavior is expected as that's one of the outcomes of enabling re-tries.

## Timeouts

Linked can limit wait times for failed outgoing requests going to another microservice. Tiemouts work by adding a key to the service profile's route configuration.

To get started, let's look at the latency for requests from `webapp` to the `books` service:

```bash
[root@osc01 ~]# linkerd -n booksapp routes deploy/webapp --to svc/books
ROUTE                     SERVICE   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
DELETE /books/{id}.json     books   100.00%   0.7rps          11ms          19ms          20ms
GET /books.json             books   100.00%   1.4rps          11ms          27ms          29ms
GET /books/{id}.json        books   100.00%   2.2rps           6ms          17ms          19ms
POST /books.json            books   100.00%   1.4rps          25ms          39ms          40ms
PUT /books/{id}.json        books   100.00%   0.7rps          71ms          97ms          99ms
[DEFAULT]                   books     0.00%   0.0rps           0ms           0ms           0ms
```

For now, lets set a timeout limit of `25ms` to the `books` service profile, and add the re-tries `timeout` property under specification.

Run the following command to edit the service profile for `books`: `kubectl -n booksapp edit sp/authors.booksapp.svc.cluster.local`

```yaml
  - condition:
      method: HEAD
      pathRegex: /authors/[^/]*\.json
    isRetryable: true
    name: HEAD /authors/{id}.json
    timeout: 25ms ### Add this line ###
```

Linkerd will now return erros to the `webapp` REST client whenever a timeout is reached. 

Timeouts includes re-try requests and maximum amount of time REST client would wait for a response.

If you run the routes command, you will see that the metrics have changed for overall effecitive and actual success rates.

```bash
[root@osc01 ~]# linkerd -n booksapp routes deploy/webapp --to svc/books -o wide
ROUTE                     SERVICE   EFFECTIVE_SUCCESS   EFFECTIVE_RPS   ACTUAL_SUCCESS   ACTUAL_RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
DELETE /books/{id}.json     books             100.00%          0.7rps          100.00%       0.7rps           5ms          19ms          20ms
GET /books.json             books             100.00%          1.4rps          100.00%       1.4rps           7ms          22ms          28ms
GET /books/{id}.json        books             100.00%          2.1rps          100.00%       2.1rps           5ms          18ms          20ms
POST /books.json            books              93.41%          1.5rps           93.41%       1.5rps          22ms          33ms          39ms
PUT /books/{id}.json        books              97.73%          0.7rps           97.73%       0.7rps          72ms          97ms          99ms
[DEFAULT]                   books               0.00%          0.0rps            0.00%       0.0rps           0ms           0ms           0ms
```

# Exporting Metrics

As of now, Linkerd only stores metrics for a 6 hour fixed window. If the metrics data is valuable, you want to store that data within a metrics store.

Internally, Linkerd stores metrics within Prometheus as a metrics store. The Prometheus's federation API is designed for copying data from one prometheus to another.

## Prometheus Federation API

Add the following under the `scraps_configs` to Prometheus config map.

```yaml
- job_name: 'linkerd'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names: ['linkerd']

  relabel_configs:
  - source_labels:
    - __meta_kubernetes_pod_container_name
    action: keep
    regex: ^prometheus$

  honor_labels: true
  metrics_path: '/federate'

  params:
    'match[]':
      - '{job="linkerd-proxy"}'
      - '{job="linkerd-controller"}'
```

```bash
[root@osc01 ~]# kubectl edit cm -n linkerd
configmap/linkerd-config skipped
configmap/linkerd-grafana-config skipped
configmap/linkerd-prometheus-config edited
```

With this update, Prometheus is now configured to federate Linkerd's metrics from Linkerd's internal Prometheus instance.

## Extracting Data using Prometheus APIs

Call the Prometheus API to extract data from Linkerd using the following command

```bash
curl -G \
  --data-urlencode 'match[]={job="linkerd-proxy"}' \
  --data-urlencode 'match[]={job="linkerd-controller"}' \
  http://linkerd-prometheus.linkerd.svc.cluster.local:9090/federate
```

Prometheus provides a JSON query API to retrieve all metrics:

```bash
[root@osc01 ~]# curl http://linkerd-prometheus.linkerd.svc.cluster.local:9090/api/v1/query?query=request_total
{"status":"success","data":{"resultType":"vector","result":[{"metric":{"__name__":"request_total","authority":"authors.booksapp.svc.cluster.local:7001","client_id":"default.booksapp.serviceaccount.identity.linkerd.cluster.local","control_plane_ns":"linkerd","deployment":"authors","direction":"inbound","instance":"10.1.230.229:4191","job":"linkerd-proxy","namespace":"booksapp","pod":"authors-86d54dd46b-jwcqc","tls":"true"},"value":[1556647157.712,"79096"]},{"metric":{"__name__":"request_total","authority":"authors.booksapp.svc.cluster.local:7001","control_plane_ns":"linkerd","deployment":"books","direction":"outbound","dst_control_plane_ns":"linkerd","dst_deployment":"authors","dst_namespace":"booksapp","dst_pod":"authors-86d54dd46b-jwcqc","dst_pod_template_hash":"86d54dd46b","dst_service":"authors","dst_serviceaccount":"default","instance":"10.1.230.245:4191","job":"linkerd-proxy","namespace":"booksapp","pod":"books-84dfdfb49b-4zq2q","server_id":"default.booksapp.serviceaccount.identity.linkerd.cluster.local","tls":"true"},
...
```

## Gather data from Linkerd Proxy directly

If users want to avoid using Prometheus, you can query Linkerd proxies directly within their `/metrics` endpoint.

View the `/metrics` within a linked proxy 

```bash
[root@osc01 ~]# kubectl -n linkerd port-forward $(kubectl -n linkerd get pods -l linkerd.io/control-plane-ns=linkerd -o jsonpath='{.items[0].metadata.name}') 4191:4191
Forwarding from 127.0.0.1:4191 -> 4191
```

And run the following `curl` command to see the output

```bash
[root@osc01 ~]# curl localhost:4191/metrics
# HELP request_total Total count of HTTP requests.
# TYPE request_total counter
request_total{direction="inbound",tls="no_identity",no_tls_reason="not_provided_by_remote"} 9723
request_total{authority="linkerd-prometheus.linkerd.svc.cluster.local:9090",direction="outbound",dst_control_plane_ns="linkerd",dst_deployment="linkerd-prometheus",dst_namespace="linkerd",dst_pod="linkerd-prometheus-84c68f7c67-f2xrb",dst_pod_template_hash="84c68f7c67",dst_service="linkerd-prometheus",dst_serviceaccount="linkerd-prometheus",tls="true",server_id="linkerd-prometheus.linkerd.serviceaccount.identity.linkerd.cluster.local"} 98373
request_total{authority="linkerd-destination.linkerd.svc.cluster.local:8086",direction="inbound",tls="true",client_id="default.emojivoto.serviceaccount.identity.linkerd.cluster.local"} 2
request_total{authority="linkerd-destination.linkerd.svc.cluster.local:8086",direction="inbound",tls="true",client_id="linkerd-prometheus.linkerd.serviceaccount.identity.linkerd.cluster.local"} 1
request_total{authority="linkerd-destination.linkerd.svc.cluster.local:8086",direction="inbound",tls="true",client_id="web.emojivoto.serviceaccount.identity.linkerd.cluster.local"} 5
request_total{authority="linkerd-destination.linkerd.svc.cluster.local:8086",direction="inbound",tls="true",client_id="default.booksapp.serviceaccount.identity.linkerd.cluster.local"} 23
request_total{authority="linkerd-destination.linkerd.svc.cluster.local:8086",direction="inbound",tls="true",client_id="linkerd-identity.linkerd.serviceaccount.identity.linkerd.cluster.local"} 1
request_total{authority="linkerd-destination.linkerd.svc.cluster.local:8086",direction="inbound",tls="true",client_id="emoji.emojivoto.serviceaccount.identity.linkerd.cluster.local"} 1
request_total{authority="linkerd-destination.linkerd.svc.cluster.local:8086",direction="inbound",tls="true",client_id="voting.emojivoto.serviceaccount.identity.linkerd.cluster.local"} 1
request_total{authority="linkerd-destination.linkerd.svc.cluster.local:8086",direction="inbound",tls="true",client_id="linkerd-web.linkerd.serviceaccount.identity.linkerd.cluster.local"} 2
request_total{authority="linkerd-controller-api.linkerd.svc.cluster.local:8085",direction="inbound",tls="true",client_id="linkerd-web.linkerd.serviceaccount.identity.linkerd.cluster.local"} 5114
request_total{direction="outbound",tls="no_identity",no_tls_reason="no_authority_in_http_request"} 263
# HELP response_latency_ms Elapsed times between a request's headers being received and its response stream completing
# TYPE response_latency_ms histogram
response_latency_ms_bucket{direction="inbound",tls="no_identity",no_tls_reason="not_provided_by_remote",status_code="200",le="1"} 7202
response_latency_ms_bucket{direction="inbound",tls="no_identity",no_tls_reason="not_provided_by_remote",status_code="200",le="2"} 8251
response_latency_ms_bucket{direction="inbound",tls="no_identity",no_tls_reason="not_provided_by_remote",status_code="200",le="3"} 8639
response_latency_ms_bucket{direction="inbound",tls="no_identity",no_tls_reason="not_provided_by_remote",status_code="200",le="4"} 8929
response_latency_ms_bucket{direction="inbound",tls="no_identity",no_tls_reason="not_provided_by_remote",status_code="200",le="5"} 9095
response_latency_ms_bucket{direction="inbound",tls="no_identity",no_tls_reason="not_provided_by_remote",status_code="200",le="10"} 9566
response_latency_ms_bucket{direction="inbound",tls="no_identity",no_tls_reason="not_provided_by_remote",status_code="200",le="20"} 9684
response_latency_ms_bucket{direction="inbound",tls="no_identity",no_tls_reason="not_provided_by_remote",status_code="200",le="30"} 9704
...
```

# Exposing the dashboard

An alternate to the Linkerd dashboard, you can exposre the dashboard metrics via an ingress. 

Lets create an ingress secret definition for nginx, this process will expose the dashboard at `dashboard.example.com` and protect it with basic auth admin/admin. 

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f 13-create-ingress-secret-nginx.yaml -n linkerd
secret/web-ingress-auth created
ingress.extensions/web-ingress created
```

Next, create an ingress secret definition for Traefik, this process will expose the dashboard at `dashboard.example.com` and protect it with basic auth admin/admin.

```bash
[root@osc01 linkerd-scripts]# kubectl apply -f 14-create-ingress-secret-traefik.yaml -n linkerd
secret/web-ingress-auth unchanged
ingress.extensions/web-ingress configured
```

# Getting Per-Route Metrics

View per-route metrics within the CLI by running the `linkerd routes` command

```bash
[root@osc01 linkerd-scripts]# linkerd routes svc/webapp -n booksapp
ROUTE                       SERVICE   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
GET /                        webapp   100.00%   0.8rps          20ms          85ms          97ms
GET /authors/{id}            webapp   100.00%   0.7rps          17ms          46ms          49ms
GET /books/{id}              webapp   100.00%   1.5rps          22ms          72ms          94ms
POST /authors                webapp   100.00%   0.7rps          19ms          39ms          40ms
POST /authors/{id}/delete    webapp   100.00%   0.8rps          27ms          77ms          96ms
POST /authors/{id}/edit      webapp     0.00%   0.0rps           0ms           0ms           0ms
POST /books                  webapp    98.85%   1.4rps          25ms          70ms          94ms
POST /books/{id}/delete      webapp   100.00%   0.7rps          16ms          28ms          30ms
POST /books/{id}/edit        webapp    97.87%   0.8rps          80ms         170ms         194ms
[DEFAULT]                    webapp     0.00%   0.0rps           0ms           0ms           0ms
```

The `[DEFAULT]` route is displayed if a route match doesn't exist the regexes specified in a service profile. Users can look up metrics by other resource types as well:

```bash
[root@osc01 linkerd-scripts]# linkerd routes deploy/webapp -n booksapp
ROUTE                       SERVICE   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
GET /                        webapp   100.00%   0.8rps          35ms          75ms          95ms
GET /authors/{id}            webapp   100.00%   0.8rps          32ms          48ms          50ms
GET /books/{id}              webapp   100.00%   1.5rps          21ms          47ms          49ms
POST /authors                webapp   100.00%   0.8rps          27ms          47ms          49ms
POST /authors/{id}/delete    webapp   100.00%   0.7rps          28ms          86ms          97ms
POST /authors/{id}/edit      webapp     0.00%   0.0rps           0ms           0ms           0ms
POST /books                  webapp    97.85%   1.6rps          23ms          80ms          96ms
POST /books/{id}/delete      webapp   100.00%   0.8rps          20ms          37ms          39ms
POST /books/{id}/edit        webapp    97.67%   0.7rps          88ms         185ms         197ms
[DEFAULT]                    webapp     0.00%   0.0rps           0ms           0ms           0ms
```

Users also have the capability to filter down to requests going from a specific resource to other services.

```bash
[root@osc01 linkerd-scripts]# linkerd routes deploy/webapp --to svc/books -n booksapp
ROUTE                     SERVICE   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
DELETE /books/{id}.json     books   100.00%   0.7rps           8ms          19ms          20ms
GET /books.json             books   100.00%   1.4rps           8ms          23ms          29ms
GET /books/{id}.json        books   100.00%   2.2rps           6ms          19ms          20ms
POST /books.json            books    95.65%   1.5rps          18ms          46ms          49ms
PUT /books/{id}.json        books    95.56%   0.8rps          71ms          97ms          99ms
[DEFAULT]                   books     0.00%   0.0rps           0ms           0ms           0ms
```

