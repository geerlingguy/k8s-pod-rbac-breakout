# Kubernetes Pod RBAC Breakout

![CI](https://github.com/geerlingguy/k8s-pod-rbac-breakout/workflows/CI/badge.svg?branch=master&event=push)

Kubernetes' Role-Based Access Control system for controlling resource permissions can be somewhat daunting to new or inexperienced users, and as such, I've seen a lot of wild and crazy clusters, granting extremely generous permissions, sometimes cluster-wide.

By default, Kubernetes is fairly secure; every namespace gets a `default` service account, which has no assigned permissions (at least as of Kubernetes 1.14), so it behaves essentially like an `unauthenticated` user.

But many apps and manifests people blindly deploy into their clusters affect the RBAC controls in ways the users may not even understand.

This project is meant to demonstrate how one particular Kubernetes visualization tool gives every pod running in the `default` namespace full cluster-admin privileges, so any pod would then be able to do almost _anything_ to almost _any_ resource in the cluster!

## Usage

> **USE AT YOUR OWN RISK**: This project doesn't intend to do anything malicious, but you've been warned.

The project assumes you're using [Minikube](https://minikube.sigs.k8s.io), because it would be stupid to test something that could potentially increase the attack surface of your cluster on a public Kubernetes cluster.

### Starting Minikube

    minikube start

### Building the Docker image

There's a Dockerfile which copies a short PHP script into a PHP/Apache container. Build it inside Minikube with the following commands:

    eval $(minikube docker-env)
    docker build -t rbac-breakout .

### Deploying the Pod

Deploy a Pod running this container, accessible via a NodePort:

    kubectl apply -f manifest.yml

### Testing if the Pod can do anything it wants

Use `minikube` to open a web browser to the URL of the Pod's service, and check the output:

    minikube service breakout

You should see that, by default, this pod can't use `kubectl` for any nefarious purposes, because the default service account has no special permissions.

## Breaking out of the Pod

There are many Kubernetes projects that have very poorly-constructed RBAC permissions; I am picking one example that I saw a year or two ago, but there are many like it. In the README, at least as of early 2020, the project recommends applying the following RBAC manifest to allow its Pods to read resources in Kubernetes' API:

    kubectl apply -f https://raw.githubusercontent.com/spekt8/spekt8/master/fabric8-rbac.yaml

The problem is, that manifest has the following ClusterRoleBinding:

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: fabric8-rbac
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

If you don't know anything about Kubernetes, you might think this is okay, because it's just giving `cluster-admin` to some `fabric8-rbac` user, right?

**Wrong**: this CRB, if applied, grants `cluster-admin` privileges to _every single resource_ in the default namespace!

So if you run the `kubectl apply` command, then refresh the PHP script page, you'll see all the commands returning info. Note that the commands being run are benign in _this_ case—but if an attacker were able to execute code on one of your containers, the attacker could just as easily run:

    kubectl get ns --no-headers=true | sed "/kube-*/d" | sed "/default/d" | awk '{print $1;}' | xargs kubectl delete ns

Boom, all your cluster's non-default namespaces are gone forever.

## Testing and Development

When debugging the index.php script, use the following command to redeploy it into Minikube quickly:

    kubectl delete pod -n rbac-breakout breakout && docker rmi -f rbac-breakout && docker build -t rbac-breakout . && kubectl apply -f manifest.yml

## Author

This project is maintained by [Jeff Geerling](https://www.jeffgeerling.com), author of [Ansible for Kubernetes](https://www.ansibleforkubernetes.com) and [Ansible for DevOps](https://www.ansiblefordevops.com).
