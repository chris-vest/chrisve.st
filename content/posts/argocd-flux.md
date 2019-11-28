+++ 
draft = true
date = 2019-11-28T10:46:21Z
title = "Migrating to ArgoCD from Flux & Flux Helm Operator"
description = "Migrating Kubernetes resource ownership from Flux to ArgoCD"
slug = "argo-cd-flux-migration"
tags = ["kubernetes","argo-cd","argocd","flux","weaveworks","flux","helm","operator"]
categories = ["tech"]
externalLink = ""
+++

# Migrating to ArgoCD from Weaveworks Flux & Flux Helm Operator

1. Add all raw manifests (i.e. not `HelmReleases`) to your repo(s) which ArgoCD is watching

2. Add references to these manifests as `Applications` in ArgoCD - do not enable pruning unless you want to play with fire; you can also disable autosync if you want in order to avoid never-ending deployment cycles if differences are found between what Flux and ArgoCD see as the state. Example application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: your-app
  # You'll usually want to add your resources to the argocd namespace.
  namespace: argocd
spec:
  # The project the application belongs to.
  project: your-project

  # Source of the application manifests
  source:
    repoURL: https://github.com/your/repo.git
    targetRevision: master # branch
    path: ./path/ # path to manifests

    # directory
    directory:
      recurse: false

  # Destination cluster and namespace to deploy the application
  destination:
    server: https://your.k8s.api
    namespace: "kube-system"

  # Sync policy
  syncPolicy:
    automated:
      # Do not enable pruning yet!
      prune: false # Specifies if resources should be pruned during auto-syncing ( false by default ).
      selfHeal: true # Specifies if partial app sync should be executed when resources are changed only in target Kubernetes cluster and no git change detected ( false by default ).

  # Ignore differences at the specified json pointers
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /metadata/labels
```

2. Make sure there are no notable differences using ArgoCD's `APP DIFF` feature - labels can mostly be ignored given the differences in how ArgoCD and Flux handle ownership - if there are differences, or errors in deploying the manifests, fix them!

3. Migrate all Flux `HelmReleases` to ArgoCD `Applications` using a Helm source - again, do not enable pruning unless you want to play with fire! Remember to [add the Helm source as an allowed source for your project](https://argoproj.github.io/argo-cd/user-guide/projects/#managing-projects). Example `Application` using a Helm source:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-ingress
  namespace: argocd
spec:
  project: your-project

  source:
    repoURL: 'https://kubernetes-charts.storage.googleapis.com'
    targetRevision: 1.6.17
    path: ''
    chart: nginx-ingress
    helm:
      releaseName: nginx-ingress
      values: |-
        controller:
          ingressClass: nginx
          publishService:
            enabled: true
          replicaCount: 3
 
  destination:
    server: https://your.k8s.api
    namespace: nginx
  
  # Sync policy
  syncPolicy:
    automated:
      # Do not enable pruning yet!
      prune: false # Specifies if resources should be pruned during auto-syncing ( false by default ).
      selfHeal: true # Specifies if partial app sync should be executed when resources are changed only in target Kubernetes cluster and no git change detected ( false by default ).

  # Ignore differences at the specified json pointers
  ignoreDifferences: []
```

4. Apply each application one-by-one, making sure there are no notable differences using ArgoCD's `APP DIFF` feature - again, labels can mostly be ignored given the differences in how ArgoCD and Flux handle ownership - if there are differences or errors in deploying the Helm charts, fix them! Especially with a Helm source, ArgoCD is much more picky than the Flux Helm Operator - ArgoCD will most likely throw errors which the Flux Helm Operator did not.

5. We can't just delete the `HelmReleases`, beceause that will cause the Flux Helm Operator to _delete_ those resources, which is not what we want. So, when we delete the `HelmReleases`, we need to make sure the Flux Helm Operator is not alive to see it. So, when everything has been checked, you can continue with the following steps:
    * Delete Flux Helm Operator deployment
    * Delete Flux deployment (in order to avoid resources being deployed again - namely the Helm operator most importantly)
    * Delete the `HelmRelease` CRD from K8s API - this will remove _all_ of the applied `HelmRelease` CRDs

6. You have now successfully migrated from Flux to ArgoCD with no downtime!