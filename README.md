# ROSA Documentation

This repository serves as the Gitops repo for the Openshift ROSA Cluster.

## Cluster remediation

If Argocd Breaks and most projects deployed in the ROSA Cluster go down, take these steps to remediate the issue:

1. Install or ensure the installation of the `Openshift-Gitops Operator`.
  - If the Openshift-Gitops operator is already correctly installed, then the cluster will have provisioned the `Openshift-Gitops` project, and you may verify this by navigating to this namespace and checking that all pods are running correctly, and you can access the `openshift-gitops-server` route provisioned in that namespace.
  - If you wish to install the operator, navigate to the `default` namespace and begin installing the operator through the `OperatorHub`.
2. Install the various Argocd `Projects`.
  - Projects in Argocd define groups of applications and the various apiVersions and types of resources that can be deployed within that project. This is why the projects needed to be created first so that argocd has the permissions to create resources of the various `kind`s deployed by each application.
  - Rosa projects are available [here](https://github.com/redhat-et/rosa-apps/tree/main/argocd/overlays/rosa/projects). Build and apply them to the `openshift-gitops` namespace in the correct directory using the folllowing command:

  ```
  cd argocd/overlays/rosa/projects
  oc project openshift-gitops
  kustomize build ./ | oc apply -f -
  ```

3. Create the App-of-apps application.
  - In our gitops repo, we used the app-of-apps pattern. This means we create one Argocd application that, in turn, deploys every other Argocd application. Our app-of-apps application is available [here](https://github.com/redhat-et/rosa-apps/tree/main/argocd/overlays/rosa/applications/app-of-apps). Once the app-of-apps application has successfully been created, it should discover every other Argocd application available [here](https://github.com/redhat-et/rosa-apps/tree/main/argocd/overlays/rosa/applications/envs/rosa), and create them as "Application Tiles" within Argocd.
  -NOTE: When migrating between Operate-First's base deployment of Argocd and the Rosa Openshift-Gitops deployment, an upgrade to the Argocd version occurred. We encountered a breaking change that Argocd could no longer use a project as a template for another project. In the old deployment, some `projects` inherited some `namespaceResourceWhitelist` values from `global-project`, which was used as a template. Since this is no longer the case, if any applications fail to deploy because of permission issues in a particular `project`, identify which `namespaceResourceWhitelist` values it is missing from the Argocd application sync-status and add them from `global-project` into the `project` in which the application is failing.
  -Note: Syncing `app-of-apps-rosa` does not sync every other application, instead updates the list of applications within the `app-of-apps-rosa`. Therefore if you have a breaking change or something is out of whack in a specific application, it will not affect anything to synchronize `app-of-apps-rosa`.
4. Sync the `cluster-resources` application.
  - Navigate to the `cluster-resources` applicaiton and sync it. This must happen before any other application is synced, with the only exceptions of the `argocd projects` and `app-of-apps-rosa` applications. The order is important because the `cluster-resources` app deploys all cluster-scoped resources which are referenced and used in the other applications, such as RBAC, namespaces, operator bundles,  etc. If the specified order is not followed, the applications may fail to sync.
5. Sync every other application.
  - Once the `Argocd-projects`, `app-of-apps-rosa`, and `cluster-resources` applications are correctly constructed, then go through each application and sync them back in to bring them back to their former state.

## Creating sealed secrets

Creating sealed secrets is pretty simple; follow these steps to do it correctly.

Pre-requisites:
- Access to the Openshift cluster and the ability to view secrets within the namespace in which you wish to create sealed secrets
- the `kubeseal` binary (see [releases](https://github.com/bitnami-labs/sealed-secrets/releases))

1. Log into the Openshift console or CLI and navigate to the project in which your secret lives.

2. Copy that Kubernetes secret to a local file. You should remove any system-generated fields (k8s or Openshift), and anything they contain. These would include, but are not limited to:
  - `ownerReferences`
  - `managedFields`
  - `creationTimestamp`
  - `resourceVersion`
  - `uid`

3. Generate your sealed secret. Firstly, ensure that you are in the correct namespace in which you wish to create your secret. You can verify this with the `oc project -q` command. After you are sure you're in the correct namespace, use the following command on the secret you just copied and edited:

```
cat <k8s_secret>.yaml | kubeseal \
    --server https://api.open-svc-sts.k1wl.p1.openshiftapps.com:6443 \
    --format yaml \
    > <k8s_secret>-sealed.yaml
```

Congratulations! You now have a properly generated sealed-secret that can be safely stored in `git`.

If encrypting your sealed-secret failed, you can take a look at the logs for the `sealed-secrets-controller` Pod available in the [kube-system namespace](https://console-openshift-console.apps.open-svc-sts.k1wl.p1.openshiftapps.com/k8s/ns/kube-system/core~v1~Pod), or contact an admin to do so if you lack the permissions.
