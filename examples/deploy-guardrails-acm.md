# Deploy a Guardrails Environment using the Anthos Config Management

## Getting Started

Our first step will be to provision a Config Controller Instance to deploy our infrastructure with.

To do that follow the steps outlined in the advanced [install guide](../docs/advanced-install.md). This process should take about 20mins.

## Setting Up ACM

Now that we have a Config Controller Instance up we'll want to create a Source Repository for us to store our infrastructure code in. For this example we will be using Cloud Source Repositories because we can quickly create it and access it using Workload Identity, so no keys to manage! If you already have a favorite source repository you should be able to use that following these [instructions](https://cloud.google.com/anthos-config-management/docs/how-to/installing-config-sync#git-creds-secret).

The following steps are modified from this [guide](https://cloud.google.com/anthos-config-management/docs/how-to/config-controller-setup#manage-resources)

0. Set Environment Variables
```
export PROJECT_ID=<config-controller-project-id>
export CONFIG_CONTROLLER_NAME=<name-of-config-controller-instance>
```

1. Enable source repository service.

```
apiVersion: serviceusage.cnrm.cloud.google.com/v1beta1
kind: Service
metadata:
  name: sourcerepo.googleapis.com
  namespace: config-control
```

2. Apply the manifest

```
kubectl apply -f service.yaml
```

This should take a few minutes for the service to enable. You can check on it's status or watch it deploy by running `kubectl wait -f service.yaml --for=condition=Ready`

3. Create the Repository Configs
```
# repo.yaml

apiVersion: sourcerepo.cnrm.cloud.google.com/v1beta1
kind: SourceRepoRepository
metadata:
  name: guardrails-configs
  namespace: config-control
```

4. Create the reposiory
```
kubectl apply -f repo.yaml
```

Now we have our base infrastructure in place and we can now set up some IAM so the ACM instance we will create can access the Source Repo.

1. Create the IAM permissions. This is grant source repository reader access to the ACM service account.
```
# gitops-iam.yaml

apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMServiceAccount
metadata:
  name: config-sync-sa
  namespace: config-control
spec:
  displayName: ConfigSync

---

apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMPolicyMember
metadata:
  name: config-sync-wi
  namespace: config-control
spec:
  member: serviceAccount:${PROJECT_ID}.svc.id.goog[config-management-system/root-reconciler]
  role: roles/iam.workloadIdentityUser
  resourceRef:
    apiVersion: iam.cnrm.cloud.google.com/v1beta1
    kind: IAMServiceAccount
    name: config-sync-sa

---

apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMPolicyMember
metadata:
  name: allow-configsync-sa-read-csr
  namespace: config-control
spec:
  member: serviceAccount:config-sync-sa@${PROJECT_ID}.iam.gserviceaccount.com
  role: roles/source.reader
  resourceRef:
    apiVersion: resourcemanager.cnrm.cloud.google.com/v1beta1
    kind: Project
    external: projects/${PROJECT_ID}
```

2. Apply the configs

```
kubectl apply -f gitops-iam.yaml
```

3. Configure the ACM instance.

```
# config-management.yaml

apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  enableMultiRepo: true
  enableLegacyFields: true
  policyController:
    enabled: true
  clusterName: krmapihost-${CONFIG_CONTROLLER_NAME}
  git:
    policyDir: configs
    secretType: gcpserviceaccount
    gcpServiceAccountEmail: config-sync-sa@${PROJECT_ID}.iam.gserviceaccount.com
    syncBranch: main
    syncRepo: https://source.developers.google.com/p/${PROJECT_ID}/r/guardrails-infra
  sourceFormat: unstructured
```

4. Deploy ACM

```
kubectl apply -f config-management.yaml
```

This will take a few minutes to deploy and while we wait for that we'll get the guardrails configuration up and running.

## Guardrails installation.

1. Get Guardrails Package.

```
kpt pkg get https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit.git/solutions/guardrails guardrails
```

This will pull down the configurations required to deploy the guardrails infrastructure.

2. Our next step will be to populate the setters file with any relevant information.

```
cd guardrails
```

Open the `setters.yaml` file in your favorite text editor. The most important fields here are at the top of the file.

3. Populate the config files with the setters information.
```
kpt fn render
```

This will populate the required fields with the informatin you set in `setters.yaml`

4. Clone the repo that you created in the previous steps.
```
gcloud source repos clone guardrails-infra
```

```
mv guardrails guardrails-infra
```

```
cd guardrails-infra
```

```
git config init.defaultBranch main
```

5. Commit your changes to git and push the configs to the repository.

```
git add .
git commit -m "Add Guardrails solution"
git push
```

After a few minutes you should start to see the resources deploying into the Config Controller instance

```
kubectl get gcp -n config-control
```

## Clean Up

1. Remove resources from git to delete them from config controller.
```
git rm -rf .
git commit -m "deleted guardrails solution"
```

2. Delete Config Controller
```
gcloud anthos config controller delete --location=us-central1 ${CONFIG_CONTROLLER_NAME}
```