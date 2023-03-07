# gitops-workloads-demo

This repository demonstrates how Helm based work loads can be managed by ArgoCD. 

## OpenShift

    $ oc adm policy add-cluster-role-to-user cluster-admin  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n openshift-gitops 

## Application setup

The configuration for each application is stored under the [apps](apps) directory. There is a [chart](apps/demo1/chart) directory to store the helm chart of the application and an [envs](apps/demo1/envs) directory to record the helm values file to be used when deploying to the "dev", "test" or "prod" environments.

    apps
    ├── demo1
        ├── chart
        │   ├── Chart.yaml
        │   └── ..
        └── envs
            ├── values-dev.yaml
            ├── values-prod.yaml
            └── values-test.yaml

## Testing the helm chart

You can test the helm chart generation as follows.

    helm dependency build apps/demo1/chart
    helm template apps/demo1/chart -f apps/demo1/envs/values-dev.yaml

## ArgoCD configuration

The ArgoCD configuration for each environment is recorded in the following files.

* [projects/dev.yaml](projects/dev.yaml)
* [projects/test.yaml](projects/test.yaml)
* [projects/prod.yaml](projects/prod.yaml)

If you investigate you'll discover each file configures two things, an [ArgoCD project](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/) 
and an [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/) to deploy the helm charts.
Assuming each cluster is running ArgoCD, you can bootstrap any of the workloads as follows:

    kubectl -n argocd apply -f projects/dev.yaml   # Run this against the "Dev" cluster

# Software

The following tool dependencies

* Azure CLI 
* kubectl
* argocd cli

# DEMO

## Setup

Provision three kubernetes clusters. The Makefile has logic to create AKS clusters on  Azure. Possible to use other mechanisms (AWS EKS, minikube, kind)

    make provision creds-cluster SUBSCRIPTION=$SUB SEQ=dev
    make provision creds-cluster SUBSCRIPTION=$SUB SEQ=test
    make provision creds-cluster SUBSCRIPTION=$SUB SEQ=prod

Perform a "core" install of ArgoCD on the k8s clusters

    kubectl --context scoil-dev create namespace argocd
    kubectl --context scoil-dev apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml

    kubectl --context scoil-test create namespace argocd
    kubectl --context scoil-test apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml

    kubectl --context scoil-prod create namespace argocd
    kubectl --context scoil-prod apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml

Bootstrap workloads onto the clusters

    kubectl --context scoil-dev  -n argocd apply -f projects/dev.yaml
    kubectl --context scoil-test -n argocd apply -f projects/test.yaml
    kubectl --context scoil-prod -n argocd apply -f projects/prod.yaml

## ArgoCD UIs

To launch UI requires running one of the following proxy sessions

Dev cluster

    kubectl config set-context scoil-dev --namespace argocd
    kubectl config use-context scoil-dev
    argocd admin dashboard --port 8081

Test Cluster

    kubectl config set-context scoil-test --namespace argocd
    kubectl config use-context scoil-test
    argocd admin dashboard --port 8082

Prod cluster

    kubectl config set-context scoil-prod --namespace argocd
    kubectl config use-context scoil-prod
    argocd admin dashboard --port 8083

The Argo CD UIs are available at following URLs:

* http://localhost:8081
* http://localhost:8082
* http://localhost:8083

## Promoting releases

This example illustrates a promotion process where docker images are promoted by being pushed into different docker registries

```mermaid
flowchart LR
   id1[(Development)] -- promote --> id2[(Test)] -- promote --> id3[(Production)]
```


### Push a release candidate to Dev

**Step 1**

Login to the Dev registry

    az acr login --name scoilmarkdev.azurecr.io

Push a pre-built image to the the Dev registry 

    docker pull nginx:1.22.0
    docker tag nginx:1.22.0 scoilmarkdev.azurecr.io/nginx:1.22.0
    docker push scoilmarkdev.azurecr.io/nginx:1.22.0

**Step 2**

Tell ArgoCD to deploy the image

    #
    # Update the image spec
    #
    export IMAGE=scoilmarkdev.azurecr.io/nginx:1.22.0
    yq e -i '.app.containers[0].image=strenv(IMAGE)' apps/demo1/envs/values-dev.yaml

    #
    # Commit and push change
    #
    git add apps/demo1/envs/values-dev.yaml
    git commit -am "Update image to $IMAGE"
    git push

### Promote release candidate to Test

**Step 1**

Push the image into the Test docker registry

    az acr import --name scoilmarktest --source scoilmarkdev.azurecr.io/nginx:1.22.0

**Step 2**

Tell ArgoCD to deploy the image

    #
    # Update the image spec
    #
    export IMAGE=scoilmarktest.azurecr.io/nginx:1.22.0
    yq e -i '.app.containers[0].image=strenv(IMAGE)' apps/demo1/envs/values-test.yaml

    #
    # Commit and push change
    #
    git add apps/demo1/envs/values-test.yaml
    git commit -am "Update image to $IMAGE"
    git push

### Promote release candidate to Prod

**Step 1**

Push the image into the Prod docker registry

    az acr import --name scoilmarkprod --source scoilmarktest.azurecr.io/nginx:1.22.0

**Step 2**

Tell ArgoCD to deploy the image

    #
    # Update the image spec
    #
    export IMAGE=scoilmarkprod.azurecr.io/nginx:1.22.0
    yq e -i '.app.containers[0].image=strenv(IMAGE)' apps/demo1/envs/values-prod.yaml

    #
    # Commit and push change
    #
    git add apps/demo1/envs/values-prod.yaml
    git commit -am "Update image to $IMAGE"
    git push


# Cleanup

The following command will delete the Azure resource group created by this tutorial 

    make purge SUBSCRIPTION=$SUB SEQ=dev
    make purge SUBSCRIPTION=$SUB SEQ=test
    make purge SUBSCRIPTION=$SUB SEQ=prod

