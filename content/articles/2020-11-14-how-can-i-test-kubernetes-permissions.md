---
title: "How can I test Kubernetes permissions?"
date: 2020-11-14T16:11:24-05:00
tags: [kubernetes,rbac]
draft: false
summary: "Or, &ldquo;impersonating a service account with <code>kubectl</code> to verify permissions are correctly configured&rdquo;"
---

They say a `bash` one-liner is worth a thousand words, or something like that...

```bash
kubectl --as system:serviceaccount:default:myapp get pods
# Error from server (Forbidden): pods is forbidden: User 
# "system:serviceaccount:default:myapp" cannot list resource 
# "pods" in API group "" in the namespace "default"
```

---

Recently I found myself setting up a [Kubernetes service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) to give my application read access to some Kubernetes resources.  Following the [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege), I wanted to ensure my application had the absolute minimum access it needed.

As I prepared the Kubernetes YAML files, I wondered how I could test these locally before deploying.  Follow along to below to see the process I used to:

1. Create a few service accounts,
2. give them access to the Kubernetes API, and
3. verify the permissions behave as expected

# Before we begin

And, as we'll be using a local Minikube cluster to test this, you'll need [`minikube`](https://minikube.sigs.k8s.io/docs/start/) installed.

We'll start off with a fresh, new minikube instance:

```sh
minikube start
# 😄  minikube v1.1.0 on darwin (amd64)
# 💡  Tip: Use 'minikube start -p <name>' to create a ...
# ...
# ...
# ...
# 🏄  Done! kubectl is now configured to use "minikube"
```

`kubectl` should now be pointing at our local Minikube cluster.  Just to verify, we can run:

```sh
kubectl config current-context
# minikube
```

Perfect!

# Meet Alice and Bob

We'll need two test subjects (pun intended), lets create two `ServiceAccount`s:

```sh
kubectl create serviceaccount alice --namespace=default
# serviceaccount/alice created

kubectl create sa bob -n default
# serviceaccount/bob created
```

As our friends Alice and Bob are well known around the cryptography community, it feels fitting we use Kubernetes Secrests resources for this demonstration.

Getting them access to Kubernetes is a two step process:

1. creating a `Role` - an object that describes a set of permissions, and
2. creating a `RoleBinding` - an object that specifies which roles apply to which subjects (service accounts, users, etc...)

First, we'll create two separate roles, `secrets-writer` and `secrets-reader`:

```sh
kubectl create role secrets-writer \
  -n default \
  --verb=get,list,watch,create,update,patch,delete \
  --resource=secrets \
  --namespace=default
# role.rbac.authorization.k8s.io/secrets-writer created

kubectl create role secrets-reader \
  --verb=get,list,watch \
  --resource=secrets \
  --namespace=default
# role.rbac.authorization.k8s.io/secrets-reader created
```

Next we'll create a couple `RoleBinding`s to give:

- Alice write access to secrets, and
- Bob read access to secrets

```sh
kubectl create rolebinding alice-secrets-writer \
  --role=secrets-writer \
  --serviceaccount=default:alice \
  --namespace=default
# rolebinding.rbac.authorization.k8s.io/alice-secrets-writer created

kubectl create rolebinding bob-secrets-reader \
  --role=secrets-reader \
  --serviceaccount=default:bob \
  --namespace=default
# rolebinding.rbac.authorization.k8s.io/bob-secrets-reader created
```

# Checking our work

Now is a good time to verify that the permissions are behaving correctly.

Lucky for us, `kubectl` has the perfect tool for the job.  From the man page:

```
    --as=""      Username to impersonate for the operation
```


