# How to Model Your Gitops Environments

This is an example on how to model Kustomize folders for a GitOps application and promote releases
between environments.

Read the full blog post. You might want to see the [application source code as well](https://github.com/kostis-codefresh/gitops-promotion-source-code).

## Folder structure

The base directory holds configuration which is common to all environments. It is not expected to change often. If you want to do changes to multiple environments at the same time it is best to use the “variants” folder.

The variants folder (a.k.a mixins, a.k.a. components) holds common characteristics between environments. It is up to you to define what exactly you think is “common” between your environments after researching your application as discussed in the previous section.

In the example application, we have variants for all prod and non-prod environments and also the regions. Here is an example of the prod variant that applies to ALL production environments.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-deployment
spec:
  template:
    spec:
      containers:
      - name: webserver-simple
        env:
        - name: ENV_TYPE
          value: "production"
        - name: PAYPAL_URL
          value: "production.paypal.com"   
        - name: DB_USER
          value: "prod_username"
        - name: DB_PASSWORD
          value: "prod_password"                     
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
```

In the example above we make sure that all production environments are using the production DB credentials, the production payment gateway and a liveness probe. These settings belong to the set of configuration that we don’t expect to promote between environments but we assume that they will be static across the application lifecycle.

With the base and variants ready we can now define every final environment with a combination of those properties.

Here is an example of the staging ASIA environment:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: staging
namePrefix: staging-asia-

resources:
- ../../base

components:
  - ../../variants/non-prod
  - ../../variants/asia

patchesStrategicMerge:
- deployment.yml
- version.yml
- replicas.yml
- settings.yml
```

First we define some common properties. We inherit all configuration from base, from non-prod environments and for all environments in Asia.

The key point here is the patches that we apply. The version.yml and replicas.yml are self-explanatory. They only define the image and replicas on their own and nothing else.

The version.yml file (which is the most important thing to promote between environments) defines only the image of the application and nothing else.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-deployment
spec:
  template:
    spec:
      containers:
      - name: webserver-simple
        image: docker.io/kostiscodefresh/simple-env-app:2.0
```

The associated settings for each release that we DO expect to promote between environments are also defined in settings.yml 

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-deployment
spec:
  template:
    spec:
      containers:
      - name: webserver-simple
        env:
        - name: UI_THEME
          value: "dark"
        - name: CACHE_SIZE
          value: "1024kb"
        - name: PAGE_LIMIT
          value: "25"
        - name: SORTING
          value: "ascending"    
        - name: N_BUCKETS
          value: "42"         
```
