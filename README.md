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

[root@osc01 linkerd-scripts]# kgrs -n sample-inject-enabled-ns
NAME                  DESIRED   CURRENT   READY   AGE
helloworld-f5b7bfc5   1         1         1       31m

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

