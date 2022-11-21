---
title: "Kubernetes resources for Traefik Proxy installation"
description: "Configure Kubernetes resource files for Traefik Proxy."
---

# Kubernetes resources for Traefik Proxy

Learn how to configure Kubernetes resource files for Traefik Proxy.

To get started, consider using the [quick start process](quick-start-with-kubernetes.md).
It's faster and gets Traefik Proxy up and running in a few minutes.

<!--
I'm not sure how relevant this document is as-is. I think the most helpful insights here are
the minimal configurations for the resources files, which they get with the sample files already.
I would consider adding the additional information that we have here directly into each file,
linking to their complete reference documentation.
For example, for the RBAC file below, add this line in the sample file itself:

See the complete [RBAC reference file](../../reference/dynamic-configuration/kubernetes-crd/#rbac).

If we do this with each resource, we can drop this document entirely.
-->

## Resources for the Traefik Dashboard

### RBAC role

This resource file creates an [RBAC role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
to grant Traefik permission to access your cluster through the Kubernetes API:

```yaml tab="traefik-rbac-role.yml"
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-role
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
```

!!! info 
 
    See the complete [RBAC reference file](../../reference/dynamic-configuration/kubernetes-crd/#rbac).

### Service Account

This resource file creates a
[service account](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/service-account-v1/#ServiceAccount)
for Traefik:

```yaml tab="traefik-service-account.yml"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-account
```

### Role binding

This resource file creates a
[role binding](https://kubernetes.io/docs/reference/kubernetes-api/authorization-resources/cluster-role-binding-v1/#ClusterRoleBinding)
to bind the RBAC role to the service account:

```yaml tab="traefik-role-binding.yml"
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-role
subjects:
  - kind: ServiceAccount
    name: traefik-account
    namespace: default # Using "default" because we did not specify a namespace when creating the ClusterAccount.
```

<!-- About code comment: shouldn't we create a namespace from the beginning? -->

### Traefik Dashboard deployment

This resource file creates a [deployment](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/)
for the Traefik Proxy Dashboard:

```yaml tab="traefik-dashboard-deployment.yml"
kind: Deployment
apiVersion: apps/v1
metadata:
  name: traefik-deployment
  labels:
    app: traefik
spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-account
      containers:
        - name: traefik
          image: traefik:v2.9
          args:
            - --api.insecure
            - --providers.kubernetesingress
          ports:
            - name: web
              containerPort: 80
            - name: dashboard
              containerPort: 8080
```

This deployment enables the Traefik Dashboard through a [static configuration](../reference/static-configuration/cli.md)
and uses Kubernetes native Ingress to route incoming requests.

You can use the attribute `args` to customize Traefik, such as:

- Enable/disable the Traefik Dashboard.
- Configure entry points.
- Select dynamic configuration providers.

<!-- ^ We should link each bullet point to its own doc. -->

Notes:

- When there is no entry point in the static configuration, Traefik creates the
default port by the `name: web` using the `containerPort` `80` to route HTTP requests.
- When you enable the [`api.insecure`](../../operations/api/#insecure) mode under `args`,
Traefik exposes the `dashboard` on the `containerPort: 8080`.

<!-- ^ I guess we don't want an "insecure api mode" in a production environment, right?
If so, I think it would be good to explain how to make this secure. -->

### Traefik services

This resource creates two [services](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/#Service) available through two pods:

- One for the Traefik Proxy Dashboard (`traefik-dashboard-service`), through the port 8080.
- One for a web application (`traefik-web-service`), through port 80.

```yaml tab="traefik-services.yml"
apiVersion: v1
kind: Service
metadata:
  name: traefik-dashboard-service
spec:
  type: LoadBalancer
  ports:
    - port: 8080
      targetPort: dashboard
  selector:
    app: traefik
---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-service
spec:
  type: LoadBalancer
  ports:
    - targetPort: web
      port: 80
  selector:
    app: traefik
```

Each `spec` represents a pod, which is required to forward the traffic to any service within the instance.

!!! warning "Configure the `spec.type`"

    Depending on your working environment and use case, you might want to change the `spec.type`.
    Make sure you understand the available [service types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) before applying this resource.

<!-- ^ I'm not sure I described this section correctly. I'd need guidance/review. -->

### Apply the resources for the Traefik Dashboard

To apply the resources above, run:

```shell
kubectl apply -f traefik-rbac-role.yml \
              -f traefik-service-account.yml \
              -f traefik-role-binding.yml \
              -f traefik-dashboard-deployment.yml \
              -f traefik-services.yml
```

## Resources for an application

To run an application behind Traefik as a reverse proxy, use an example
([`hello-world`](https://github.com/VirtuaCreative/html-hello-world)) to get started.

The `hello-world` application is a simple webpage on an HTTP server running on port 80.
Its image is available on Docker Hub under [`ramosmd/html-hello-world`](https://hub.docker.com/repository/docker/ramosmd/html-hello-world/).

You can use a similar configuration to deploy any other application.

To set Traefik as as a reverse-proxy for your application, you need to
create the resources below.

### Application deployment

This resource creates a deployment for your application:

```yaml tab="traefik-app-deployment.yml"
kind: Deployment
apiVersion: apps/v1
metadata:
  name: hello-world
  labels:
    app: hello-world

spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: ramosmd/html-hello-world:v2
          ports:
            - name: web
              containerPort: 80
```

### Application service

This resource creates a service for the deployment just created:

```yaml tab="traefik-app-services.yml"
apiVersion: v1
kind: Service
metadata:
  name: hello-world

spec:
  ports:
    - name: web
      port: 80
      targetPort: web
      
  selector:
    app: hello-world
```

### Application Ingress

This resource creates an Ingress for the application:

```yaml tab="traefik-app-ingress.yml"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              name: web
```

With this configuration, Traefik redirects any incoming requests
starting with `/` to the web service.

Traefik is notified when an Ingress resource is created, updated,
or deleted. This is known as Traefik's  [dynamic configuration](../../providers/kubernetes-ingress/).

### Apply the resources for the web application

To apply the application resources above, run:

```shell
kubectl apply -f traefik-app-deployment.yml \
              -f traefik-app-services.yml \
              -f traefik-app-ingress.yml
```

Both the dashboard and the application are available on your cluster:

- To access the Traefik Proxy Dashboard on your browser, visit [`http://localhost:8080`](http://localhost:8080).
- To access the `hello-world` application, run `curl -v http://localhost/`.

<!-- ^ None of these work for me though (on Minikube). -->

!!! tip

    Find more information on [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/),
    and [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) in the official Kubernetes documentation.

!!! question "Going further"

    - [Filter Ingresses](../providers/kubernetes-ingress.md#ingressclass) to use with [IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class)
    - Use [IngressRoute CRD](../providers/kubernetes-crd.md)
    - Protect [Ingresses with TLS](../routing/providers/kubernetes-ingress.md#enabling-tls-via-annotations)

<!--  ^ For the documentation linked above, I'd rather place them in context,
  where they are meaningful to the user, rather than detached at the end of the document. -->
