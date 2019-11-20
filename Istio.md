# Istio Tutorial
## Setup Istio

1. I'm using Minishift to setup openshift/kuberenetes cluster. 
2. For Virtualization I'm using Virtual Box.

### Install Minishift
Follow this [link][miinstall] for instructions to install minishift. Once it is installed we need to setup minishift for installing istio cluster.

Following command sequence creates a new openshift cluster using minishift. We are creating a profile `istio-tutorial` to setup istio.

``` bash
# add the location of minishift executable to PATH
# I also keep other handy tools like kubectl and kubetail.sh
# in that directory

minishift profile set istio-tutorial
minishift config set memory 8GB
minishift config set cpus 3
minishift config set image-caching true
minishift config set openshift-version v3.11.0
minishift addon enable admin-user
# Since we are using virtualbox
minishift config set vm-driver virtualbox

#cdk 3.7 bug - docker url check
minishift config set skip-startup-checks true

minishift start
#This needs to be executed again if you restart minishift.
minishift ssh -- sudo setenforce 0

# Openshift console bug. anyuid needs to be applied after startup
minishift addon apply anyuid
```

Once openshift cluster is up use following commands to access oc and docker commands from the terminal

``` bash
eval $(minishift oc-env)
eval $(minishift docker-env)
oc login $(minishift ip):8443 -u admin -p admin
```

Now you need to prepare your OpenShift/Kubernetes environment. Istio uses ValidatingAdmissionWebhook for validating Istio configuration and MutatingAdmissionWebhook for automatically injecting the sidecar proxy into user pods. Update Minishift’s default configuration by running the following:

``` bash
minishift openshift config set --target=kube --patch '{
    "admissionConfig": {
        "pluginConfig": {
            "ValidatingAdmissionWebhook": {
                "configuration": {
                    "apiVersion": "v1",
                    "kind": "DefaultAdmissionConfig",
                    "disable": false
                }
            },
            "MutatingAdmissionWebhook": {
                "configuration": {
                    "apiVersion": "v1",
                    "kind": "DefaultAdmissionConfig",
                    "disable": false
                }
            }
        }
    }
}'
```

Use `minishift console` command to open a browser window with minishift console. Defaule username/password is admin/admin.

### Install Istio
#### Download Istio
``` bash
#!/bin/bash

# Mac OS:
curl -L https://github.com/istio/istio/releases/download/1.3.5/istio-1.3.5-osx.tar.gz | tar xz

# Fedora/RHEL:
curl -L https://github.com/istio/istio/releases/download/1.3.5/istio-1.3.5-linux.tar.gz | tar xz
```

#### Install Istio in minishift

Following command deploys istio into the cluster (`istio-system` namespace) and creates a new project `istio-system` in minishift console
``` bash
cd istio-1.3.5 # Downloaded Istio directory
export ISTIO_HOME=`pwd`
export PATH=$ISTIO_HOME/bin:$PATH

for i in install/kubernetes/helm/istio-init/files/crd*yaml; do oc apply -f $i; done

oc apply -f install/kubernetes/istio-demo.yaml
oc project istio-system

oc expose svc istio-ingressgateway --port=80
oc expose svc grafana
oc expose svc prometheus
oc expose svc tracing
oc expose service kiali --path=/kiali
oc adm policy add-cluster-role-to-user admin system:serviceaccount:istio-system:kiali-service-account -z default

```

Verify that istio is setup correctly
```bash
$ oc get pods -w

NAME                                        READY     STATUS      RESTARTS   AGE
grafana-55cd86b44c-2vndc                  1/1     Running     0          88m
istio-citadel-f9fbdd9df-xzzr7             1/1     Running     0          88m
istio-cleanup-secrets-1.1.6-d5css         0/1     Completed   0          88m
istio-egressgateway-895fb885d-bdqkv       1/1     Running     0          89m
istio-galley-5797db85b8-4866m             1/1     Running     0          89m
istio-grafana-post-install-1.1.6-6dk5h    0/1     Completed   0          89m
istio-ingressgateway-58f959476f-82zsf     1/1     Running     0          89m
istio-pilot-57d4bb58ff-tt8r4              2/2     Running     0          88m
istio-policy-79b88bcdf9-qqp4r             2/2     Running     6          88m
istio-security-post-install-1.1.6-8mmxj   0/1     Completed   0          88m
istio-sidecar-injector-7698fc57fb-dlnx4   1/1     Running     0          88m
istio-telemetry-b9799c89-d94hj            2/2     Running     6          88m
istio-tracing-7454db9d79-9qwqr            1/1     Running     0          88m
kiali-66d74fc6cc-zdzzt                    1/1     Running     0          88m
prometheus-7d9fb4b69c-ww5w7               1/1     Running     0          88m
```

Istio is now up. we can deploy microservices that would use istio

### Deploy Microservices
All microservices are taken from [istio-tutorial][istiod]

Create new project name `tutorial`
```bash
oc new-project tutorial
oc adm policy add-scc-to-user privileged -z default -n tutorial
```

Download the istio-tutorial code
``` bash
git clone https://github.com/redhat-developer-demos/istio-tutorial
cd istio-tutorial
```

Make sure `istioctl` is present in $PATH before depolying services

#### Deploy Customer service
```bash
oc apply -f <(istioctl kube-inject -f customer/kubernetes/Deployment.yml) -n tutorial
oc create -f customer/kubernetes/Service.yml -n tutorial
oc create -f customer/kubernetes/Gateway.yml -n tutorial
```

wait for customer service to come up `oc get pods -w -n tutorial`

Once all pods are up use curl command to check if service is running 
``` bash
$curl istio-ingressgateway-istio-system.$(minishift ip).nip.io/customer
customer => UnknownHostException: preference
```

An **UnknownHostException: preference** would be displayed. this is because other microservices (preference and recommendation) are not deployed yet.

Note: you can use stern to check the logs of the microservice `stern $(kubectl get pods|grep customer|awk '{ print $1 }'|head -1) -c customer`

#### Deploy Preference Service
```bash
oc apply -f <(istioctl kube-inject -f preference/kubernetes/Deployment.yml) -n tutorial
oc create -f preference/kubernetes/Service.yml -n tutorial
```

#### Deploy Recommendation service
```bash
oc apply -f <(istioctl kube-inject -f recommendation/kubernetes/Deployment.yml)  -n tutorial
oc create -f recommendation/kubernetes/Service.yml  -n tutorial
```

Once all services are up use the same curl command. It will give a success response
```bash
$curl istio-ingressgateway-istio-system.$(minishift ip).nip.io/customer
customer => preference => recommendation v1 from '99634814-sf4cl': 1
```

## References
1. Introducing Istio Service Mesh for Microservices, 2nd Edition - Chapter 2 Installation and Getting Started
2. [Istio Setup][istios]

[miinstall]: https://docs.okd.io/latest/minishift/getting-started/installing.html
[istios]: https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.3.x/index.html
[istiod]: https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.3.x/2deploy-microservices.html
