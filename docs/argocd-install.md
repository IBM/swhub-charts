## Disclaimer

The feature and procedures described below are from a tech preview, for fresh install as a proof of concept (POC). It is not intended for production environment, nor does it come with support for upgrade to a future release version.

## Intro

This document will go through step by step on how to deploy IBM Software Hub and Planning Analytics using ArgoCD, leveraging helm charts. The release version is 5.2.0.
You will need an Openshift cluster with valid storage classes and openshift-gitops, and enable Red Hat Cert Manager on it (https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift#cert-manager-operator-install). Also, acquire argocd cli tool here https://argo-cd.readthedocs.io/en/stable/getting_started/#2-download-argo-cd-cli.

## Air Gap

If your cluster is in air gapped environment, complete the steps listed in `image-mirror.md` file to mirror images. Then clone/download all charts from IBM helm chart repo [URL pending], and host the charts in helm repo reachable from your ArgoCD instance.

## Procedure

1. Configure your ArgoCD instance to incorporate custom health checks for IBM Software Hub and Planning Analytics. In the same directory as this document, a `custom-health-checks.yaml` file includes all health check. See https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#way-1-define-a-custom-health-check-in-argocd-cm-configmap for futher instructions.
2. Connect your argocd cli to the ArgoCD instance. See https://argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd_login/ for defferent opetions.

   ```
   ADMIN_PASSWD=$(oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
   SERVER_URL=$(oc get routes openshift-gitops-server -n openshift-gitops -o jsonpath='{.status.ingress[0].host}')
   argocd login --username admin --password ${ADMIN_PASSWD} ${SERVER_URL} --grpc-web --insecure
   ```
   or
   ```
   oc project $argocdNS
   argocd login --core
   # Note: DO NOT change oc project when using argocd cli. This will log argocd cli out.
   ```



3. Set up env vars
   ```bash
   #setup variables
   operatorNS=<operator namespace>
   instanceNS=<operand namespace>
   licensingNS=<ibm licensing service namespace>
   appSuffix=my-app #this string is added to all applications created in this doc
   fileStorageClass=<file-sc>
   blockStorageClass=<block-sc>
   imagePullPrefix=<private-registry-url>
   imagePullSecret=ibm-entitlement-key #update this value if the pull secret is not the same
   argocdNS=<openshift gitops namespace> #default is openshift-gitops
   helmRepoURL=<helm repo url> #default is https://raw.githubusercontent.com/IBM/swhub-charts/refs/heads/poc/5.2.0 if using this repo
   installConfigImageDigest="sha256:f97f1b364a27acfe08b1c80d9ce91c25f07e6005a7ba821abf164c8e38925a04"
   ```
4. Connect ArgoCD to the helm chart repo. See https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/.

   ```bash
   argocd repo add $helmRepoURL --name local-charts --username <w3@ibm.com> --password xxxxx --type helm
   ```

5. Log in to the cluster

   ```bash
   oc login -u $user -p $password --server=<your cluster>
   ```

6. Create namespaces

   ```bash
   oc new-project $operatorNS
   oc new-project $instanceNS
   oc new-project $licensingNS
   oc project $argocdNS #switch back
   ```

7. Grant ArgoCD access by binding admin role to `system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller` service account in those namespaces above. Or, grant cluster admin access to it.

8. Create an image pull secret `ibm-entitlement-key` in the operator and instance namespace and licensing namespace. Make sure the credentials can pull images from your private registry.

9. Deploy cluster-scoped apps

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: cpd-platform-cluster-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 5.2.0+20250516.013559.108
       chart: cpd-platform-cluster-scoped
       helm:
         valuesObject:
           global:
             operatorNamespace: $operatorNS
     project: default
   EOF
   argocd app sync cpd-platform-cluster-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: cs-cluster-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 4.13.0+20250527.133754.0
       chart: ibm-common-service-operator-cluster-scoped
       helm:
         valuesObject:
           global:
             operatorNamespace: $operatorNS
     project: default
   EOF
   argocd app sync cs-cluster-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: im-cluster-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 4.12.0+20250527.133754.0
       chart: ibm-iam-operator-cluster-scoped
       helm:
         valuesObject:
           global:
             operatorNamespace: $operatorNS
     project: default
   EOF
   argocd app sync im-cluster-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: planning-analytics-cluster-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 5.2.0+20250604.210250.1608
       chart: planning-analytics-cluster-scoped
       helm:
         valuesObject:
           global:
             operatorNamespace: $operatorNS
     project: default
   EOF
   argocd app sync planning-analytics-cluster-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: postgresql-cluster-${appSuffix}
     namespace: $argocdNS
   spec:
     syncPolicy:
       syncOptions:
       - ServerSideApply=true
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 5.16.0+20250422.134042.20
       chart: postgresql-cluster-scoped
       helm:
         valuesObject:
           global:
             operatorNamespace: $operatorNS
     project: default
   EOF
   argocd app sync postgresql-cluster-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: zen-cluster-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 6.2.0+20250530.152516.232
       chart: zen-cluster-scoped
       helm:
         valuesObject:
           global:
             operatorNamespace: $operatorNS
     project: default
   EOF
   argocd app sync zen-cluster-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: ibm-licensing-cluster-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 4.2.16+20250604.153216.0
       chart: ibm-licensing-cluster-scoped
       helm:
         valuesObject:
           global:
             licenseAccept: true
             imagePullPrefix: $imagePullPrefix
             imagePullSecret: $imagePullSecret
           ibmLicensing:
             namespace: $licensingNS
             watchNamespace: $licensingNS,$operatorNS,$instanceNS  # add tetheredNS if any
     project: default
   EOF
   argocd app sync ibm-licensing-cluster-${appSuffix}
   ```

10. Deploy namespaced apps (service apps)

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: cpd-platform-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 5.2.0+20250516.013559.108
       chart: cpd-platform
       helm:
         valuesObject:
           global:
             licenseAccept: true
             releaseVersion: 5.2.0
             operatorNamespace: $operatorNS
             instanceNamespace: $instanceNS
             blockStorageClass: $blockStorageClass
             fileStorageClass: $fileStorageClass
             imagePullPrefix: $imagePullPrefix
             imagePullSecret: $imagePullSecret
             installConfigImageDigest: $installConfigImageDigest
             installConfigImageName: cpopen/cpd/olm-utils-v3
     project: default
   EOF
   argocd app sync cpd-platform-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: cs-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 4.13.0+20250527.133754.0
       chart: ibm-common-service-operator
       helm:
         valuesObject:
           global:
             licenseAccept: true
             releaseVersion: 5.2.0
             operatorNamespace: $operatorNS
             instanceNamespace: $instanceNS
             blockStorageClass: $blockStorageClass
             fileStorageClass: $fileStorageClass
             imagePullPrefix: $imagePullPrefix
             imagePullSecret: $imagePullSecret
     project: default
   EOF
   argocd app sync cs-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: im-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 4.12.0+20250527.133754.0
       chart: ibm-iam-operator
       helm:
         valuesObject:
           global:
             licenseAccept: true
             operatorNamespace: $operatorNS
             instanceNamespace: $instanceNS
             tetheredNamespaces: []
             releaseVersion: 5.2.0

             blockStorageClass: $blockStorageClass
             fileStorageClass: $fileStorageClass

             imagePullPrefix: $imagePullPrefix
             imagePullSecret: $imagePullSecret
             installConfigImageDigest: $installConfigImageDigest
             installConfigImageName: cpopen/cpd/olm-utils-v3

     project: default
   EOF
   argocd app sync im-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: postgresql-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 5.16.0+20250422.134042.20
       chart: postgresql
       helm:
         valuesObject:
           global:
             licenseAccept: true
             operatorNamespace: $operatorNS
             instanceNamespace: $instanceNS
             blockStorageClass: $blockStorageClass
             fileStorageClass: $fileStorageClass
             imagePullPrefix: $imagePullPrefix
             imagePullSecret: $imagePullSecret
             installConfigImageDigest: $installConfigImageDigest
             installConfigImageName: cpopen/cpd/olm-utils-v3
           postgresql:
             entitledImagePullPrefix: $imagePullPrefix
     project: default
   EOF
   argocd app sync postgresql-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: zen-${appSuffix}
     namespace: $argocdNS
   spec:
     ignoreDifferences: #optional, ask argocd to ignore annotation and label differences
       - kind: "*"
         group: "*"
         jsonPointers:
           - /metadata/annotations
           - /metadata/labels
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 6.2.0+20250517.171311.226
       chart: zen
       helm:
         valuesObject:
           global:
             licenseAccept: true
             operatorNamespace: $operatorNS
             instanceNamespace: $instanceNS
             blockStorageClass: $blockStorageClass
             fileStorageClass: $fileStorageClass
             imagePullPrefix: $imagePullPrefix
             imagePullSecret: $imagePullSecret
             installConfigImageDigest: $installConfigImageDigest
             installConfigImageName: cpopen/cpd/olm-utils-v3
             releaseVersion: 5.2.0
     project: default
   EOF
   argocd app sync zen-${appSuffix}
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: planning-analytics-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 5.2.0+20250604.210250.1608
       chart: planning-analytics
       helm:
         valuesObject:
           global:
             licenseAccept: true
             operatorNamespace: $operatorNS
             instanceNamespace: $instanceNS
             releaseVersion: 5.2.0
             tetheredNamespaces: []
             blockStorageClass: $blockStorageClass
             fileStorageClass: $fileStorageClass
             imagePullPrefix: $imagePullPrefix
             imagePullSecret: $imagePullSecret
             installConfigImageDigest: $installConfigImageDigest
             installConfigImageName: cpopen/cpd/olm-utils-v3
           planning_analytics:
             non_olm: true
             crVersion: 5.2.0 # same as CPD release version
             imagePullSubDir: "cp/cpd"
             operatorImageName: cpopen/ibm-planning-analytics-operator # including the sub folder in which the image is saved
             operatorImageDigest:
               amd64: sha256:d4eb8e4ba47f20163d5c48fa1c60b582ba5b649da6c4b3884c366762d22c3c01
               ppc64le: "" # supporting multi-arch here
               s390x: ""
             tm1:
               operatorImageName: cpopen/ibm-planning-analytics-tm1-operator
               operatorImageDigest:
                 amd64: sha256:d4e5cf3f8a6348e53e43dbbb8caf47ae44a8ee8c5e8b0f1b7d12d9d328b2d1a7
                 ppc64le: "" # supporting multi-arch here
                 s390x: ""
             pass:
               operatorImageName: cpopen/ibm-planning-analytics-spreadsheet-operator
               operatorImageDigest:
                 amd64: sha256:5cebbaa872bb38df5ed98d6d7df2b6142f8ae0281ec470e7cfc66a1af72bc067
                 ppc64le: "" # supporting multi-arch here
                 s390x: ""
             catalogSourceName: "ibm-planning-analytics-operator-catalog" # if catalogsource name is defined, will clean up the catalogsource created by OLM
             subscriptionName: "ibm-planning-analytics-subscription" # if subscription name is defined, will clean up the subscription and csv in operator-ns created by OLM
             serviceAccountName: "pa-operator-sa"
             operandRequestName: ""
     project: default
   EOF
   argocd app sync planning-analytics-${appSuffix}
   
   oc patch serviceaccount pa-operator-sa -p "{\"imagePullSecrets\": [{\"name\": \"$imagePullSecret\"}]}" -n $operatorNS
   ```

   ```bash
   oc apply -f - << EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: platform-config-${appSuffix}
     namespace: $argocdNS
   spec:
     destination:
       name: ''
       namespace: default
       server: https://kubernetes.default.svc
     source:
       path: ''
       repoURL: $helmRepoURL
       targetRevision: 5.2.0+20250402.154803.5
       chart: platform-config
       helm:
         valuesObject:
           global:
             licenseAccept: true
             releaseVersion: 5.2.0
             operatorNamespace: $operatorNS
             instanceNamespace: $instanceNS
             imagePullPrefix: $imagePullPrefix
             imagePullSecret: $imagePullSecret
             installConfigImageDigest: $installConfigImageDigest
             installConfigImageName: cpopen/cpd/olm-utils-v3
             components:
                 - planning_analytics
                 - cpd_platform
           commands:
             applyEntitlement:
               - entitlement: cpd-enterprise
                 production: "false"
     project: default
   EOF
   argocd app sync platform-config-${appSuffix}
   ```

## Verification
At this point, you may monitor the progress using `argocd app list` or through ArgoCD's webconsole on the apps created.

At this point, all apps should at least start sync-ing (cluster scoped ones are probably done by now). Wait until all apps are "Synced" and health, and confirm all CRs are in good state (i.e. `Completed`)

## Known Issue
1. Zen app status is in `outOfSync` due to zenservice CR being overridden by the operator. No functional impact.
```
NAME                                               CLUSTER                         NAMESPACE  PROJECT  STATUS     HEALTH   SYNCPOLICY  CONDITIONS                REPO                                                                        PATH  TARGET
openshift-gitops/zen-atrva                         https://kubernetes.default.svc  default    default  OutOfSync  Healthy  Manual      <none>                    https://raw.github.ibm.com/IBMSoftwareHub/charts/refs/heads/5.2.0/promoted        6.2.0+20250517.171311.226
```

2. PA app status is in `outOfSync` because the workaround patch command was applied after app/service creation. No functional impact.
```
NAME                                               CLUSTER                         NAMESPACE  PROJECT  STATUS     HEALTH   SYNCPOLICY  CONDITIONS                REPO                                                                        PATH  TARGET
openshift-gitops/planning-analytics-atrva          https://kubernetes.default.svc  default    default  OutOfSync  Healthy  Manual      <none>                    https://raw.github.ibm.com/IBMSoftwareHub/charts/refs/heads/5.2.0/promoted        5.2.0+20250604.210250.1608
```
