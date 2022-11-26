# Creating a fully-managed Kubernetes GitOps platform with Argo CD

In this hour-long hands-on lab, you will enter a scenario as a Kubernetes Cluster Administrator. In your organization, the developers deploy containerized applications using Helm charts. The charts are version controlled in a GitHub repo, and the developers deploy them manually by running `helm` commands against the cluster.

This lab will walk you through implementing Argo CD to manage the deployment of the Helm charts in a declarative fashion.

Ultimately, you will have a Kubernetes cluster, with Applications deployed using an Argo CD control plane.

Use these links to learn what is: [Kubernetes](https://youtu.be/4ht22ReBjno?t=164), [GitOps](https://opengitops.dev/), [Argo CD](https://argo-cd.readthedocs.io/en/stable/#what-is-argo-cd), [Akuity](https://akuity.io/).

[If you prefer to learn by following along, check out the webinar where I walk through these steps.](https://www.youtube.com/watch?v=73nufrbL3Rw)

- [1. Lab Scenario](#1-lab-scenario)
- [2. Prerequisites](#2-prerequisites)
- [3. Setting up your environment.](#3-setting-up-your-environment)
  - [3.1. Create the repository from a template.](#31-create-the-repository-from-a-template)
  - [3.2. Create a Kubernetes cluster using `kind`.](#32-create-a-kubernetes-cluster-using-kind)
  - [3.3. Launch an Argo CD instance.](#33-launch-an-argo-cd-instance)
  - [3.4. Deploy an agent to the cluster.](#34-deploy-an-agent-to-the-cluster)
- [4. Using Argo CD to Deploy Helm Charts.](#4-using-argo-cd-to-deploy-helm-charts)
  - [4.1. Create an Application in Argo CD.](#41-create-an-application-in-argo-cd)
  - [4.2. Syncing changes manually](#42-syncing-changes-manually)
  - [4.3. Enable auto-sync and self-heal for the guestbook Application.](#43-enable-auto-sync-and-self-heal-for-the-guestbook-application)
  - [4.4. Demonstrate Application auto-sync via Git.](#44-demonstrate-application-auto-sync-via-git)
  - [4.5. Demonstrate Application self-heal functionality.](#45-demonstrate-application-self-heal-functionality)
- [5. Managing Argo CD Applications declaratively.](#5-managing-argo-cd-applications-declaratively)
  - [5.1. Create an App of Apps.](#51-create-an-app-of-apps)
- [6. Review.](#6-review)

## 1. Lab Scenario

To prepare for the lab scenario, you can think of what to name your *organization* and *environment*. For example, your *organization* could be your GitHub username and your *environment* “staging”.

- You will use the *organization* name for creating an organization while setting up your account on the Akuity Platform.
- You will use the *environment* name for the cluster and the App of Apps Application in Argo CD.

Text between less-than and greater-than symbols (i.e., `<...>`) indicates that you must substitute it with the value relevant to your scenario. For example, in the text `https://github.com/<github-username>/managed-argocd-lab`, you will replace `<github-username>` with your GitHub username.

## 2. Prerequisites
The lab requires that you have the following:
- a [GitHub](https://github.com/) Account - You will use this to create the repo for the environment configuration and for creating an account on the [Akuity Platform](https://akuity.io/akuity-platform/).

- the Kubernetes command-line tool, [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) - Will be used for interacting with the cluster directly.

- a Kubernetes cluster with Cluster Admin access (to create namespaces and cluster roles) or;

- [Docker Desktop](https://duckduckgo.com/?q=docker+desktop+install) and [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed on your local machine.

- [egress traffic](https://www.webopedia.com/definitions/egress-traffic/) allowed from the Kubernetes cluster to the internet.

The lab was built and tested using the following tool and component versions:
- Argo CD: v2.5.1

- Docker Desktop: 4.13.1

- Kubernertes: v1.25.2

- kind: v0.16.0

## 3. Setting up your environment.
### 3.1. Create the repository from a template.
In this scenario, the developers manage the application Helm charts in version control. To represent this in the lab, you will create a repository from a template containing the application Helm charts.

1. Click [this link](https://github.com/morey-tech/managed-argocd-lab/generate) or click "Use this template" from the repo main page.

2. Ensure the desired "Owner" is selected (e.g., your account and not an org).

3. Enter a `managed-argocd-lab` for the "Repository name". 

4. Then click "Create repository from template".

Next, you have a couple of files to update:

1. For the `guestbook.yaml` and `portal.yaml` files in `apps/`:

2. Fix the `spec.source.repoURL` by replacing `<github-username>` with your GitHub username (or the org name if using one).

3. Fix the `spec.destination.name` by replacing `<environment-name>` with your environment name.

4. Commit the changes to the `main` branch.

```diff
  spec:
    project: default
    source:
--    repoURL: 'https://github.com/<github-username>/managed-argocd-lab'
++    repoURL: 'https://github.com/morey-tech/managed-argocd-lab'
      ...
    destination:
      namespace: guestbook
--    name: <environment-name> # Update this value.
++    name: staging
```

### 3.2. Create a Kubernetes cluster using `kind`.
To deploy the application Helm charts during the lab, you will need a cluster. You could skip this section if you brought a cluster to the lab. Otherwise, you will create one on your local machine using `kind`.

1. Create a cluster using `kind` and specify your *environment* name.
   ```
   kind create cluster --name <environment-name>
   ```

   You should see the following output:
   ```
   Creating cluster "<environment-name>" ...
   ✓ Ensuring node image (kindest/node:v1.25.2) 🖼
   ✓ Preparing nodes 📦 📦 📦 📦  
   ✓ Writing configuration 📜 
   ✓ Starting control-plane 🕹️ 
   ✓ Installing CNI 🔌 
   ✓ Installing StorageClass 💾 
   ✓ Joining worker nodes 🚜 
   Set kubectl context to "kind-<environment-name>"
   You can now use your cluster with:

   kubectl cluster-info --context kind-<environment-name>

   Thanks for using kind! 😊
   ```

2. Check that the cluster works by running `kubectl get nodes`.
   ```
   % kubectl get nodes
   NAME                               STATUS   ROLES           AGE   VERSION
   <environment-name>-control-plane   Ready    control-plane   74s   v1.25.2
   ```

   Fetching the nodes will demonstrate that `kubectl` can connect to the cluster and query the API server. The node should be in the "Ready" status.

### 3.3. Launch an Argo CD instance.
This scenario demonstrates deploying applications to a cluster external to Argo CD. For convenience during the lab, you utilize the Akuity Platform to get a fully-featured instance of Argo CD without having to configure a separate cluster and set up UI access.

You can use the open-source installation of Argo CD for the lab by standing up a second cluster and following [the Getting Started guide](https://argo-cd.readthedocs.io/en/stable/getting_started/). Then [setting up an ingress](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/) and substituting section 3.4 with [connecting the first cluster](https://argo-cd.readthedocs.io/en/stable/getting_started/#5-register-a-cluster-to-deploy-apps-to-optional).

Similar to how the GitHub repo is hosting the Helm charts, which describe **what** resources to deploy into Kubernetes, the Akuity Platform will host the Application manifests, which represent **how** to deploy the resources into the cluster. Along with Argo CD, which will implement the changes on the cluster.

1. Create an account on the [Akuity Platform](https://akuity.io/signup).

2. To log in with GitHub SSO, click "Continue with GitHub".
   
   You can also use Google SSO or an email and password combo. For the sake of the lab, I'm assuming you will be using GitHub.

3. Click "Authorize akuityio".

4. Create an organization by clicking "create or join" in the information bulletin.

5. In the top right, click "New organization".

6. Enter your "Organization Name" and click "Create".

7. At the top of the sidebar, click "Argo CD".

8. In the top right, click "Create".

9. Set the "Instance Name" to `cluster-manager`.

10. Under the "Version" section, click the option corresponding to `v2.5`.

11. Click "Create".

    At this point, your Argo CD instance will begin initializing. The start-up typically takes under 2 minutes.


While the instance is initializing, you can prepare it for the rest of the lab.

1. In the dashboard for the Argo CD instance, click "Settings".

2. In the "General" section, find "Declarative Management" and enable it by clicking the toggle.
    
    Declarative Management enables using the Akuity Platform with the App of Apps pattern (or ApplicationSets) by exposing the `in-cluster` destination. Using the `in-cluster` destination means an Application can create other Applications in the cluster hosting Argo CD. You will work on this later in the lab. The open-source installation of Argo CD exposes this by default.

3. In the top right, click "Save".

4. On the inner sidebar, under "Security & Access", click "System Accounts".

5. Enable the "Admin Account" by clicking the toggle and clicking "Confirm" on the prompt.

6. Then, for the `admin` user, click "Set Password".

    On the open-source installation of Argo CD, you would retrieve this from [the `argocd-initial-admin-secret` Secret](https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli).

7. To get the password, click "Copy".
    
    If the copy option does not appear, click "Regenerate Password" and then "Copy". Note this will invalidate any other sessions for the `admin` user.

8. In the bottom right of the Set password prompt, hit "Close".

9. In the top, next to the Argo CD instance name and status, click the instance URL (e.g., `<uuid>.cd.akuity.cloud`) to open the Argo CD login page in a new tab.

10. Enter the username `admin` and the password copied previously.

You now have a fully-managed Argo CD instance.

### 3.4. Deploy an agent to the cluster.
You must connect the cluster to Argo CD to deploy the application resources. The Akuity Platform uses an agent-based architecture for connecting external clusters. So, you will provision an agent and deploy it to the cluster.

1. Back on the Akuity Platform, in the top left of the dashboard for the Argo CD instance, click "Clusters".

2. In the top right, click "Connect a cluster".

3. Enter your *environment* name as the "Cluster Name".

4. In the bottom right, Click "Connect Cluster".

5. To get the agent install command, click "Copy to Clipboard". Then, in the bottom right, "Done".

6. Open your terminal and check that your target is the correct cluster by running `kubectl config current-context`. 
    
    If you are following along using `kind`, you should see the following:
    ```
    % kubectl config current-context
    kind-<env-name>
    ```

7. Paste and run the command against the cluster.
  
  The command will create the `akuity` namespace and deploy the resources for the Akuity Agent.

8. Check the pods in the `akuity` namespace. Wait for the `Running` status on all pods (approx. 1 minute).
   ```
   % kubectl get pods -n akuity
   NAME                                                        READY   STATUS    RESTARTS   AGE
   akuity-agent-<replicaset-id>-<pod-id>                       1/1     Running   0          65s
   akuity-agent-<replicaset-id>-<pod-id>                       1/1     Running   0          65s
   argocd-application-controller-<replicaset-id>-<pod-id>      2/2     Running   0          65s
   argocd-notifications-controller-<replicaset-id>-<pod-id>    1/1     Running   0          65s
   argocd-redis-<replicaset-id>-<pod-id>                       1/1     Running   0          65s
   argocd-repo-server-<replicaset-id>-<pod-id>                 1/1     Running   0          64s
   argocd-repo-server-<replicaset-id>-<pod-id>                 1/1     Running   0          64s
   ```
   
   Re-run the `kubectl get pods -n akuity` command to check for updates on the pod statuses.

9. Back on the Clusters dashboard, confirm that the cluster shows a green heart before the name, indicating a healthy status.

## 4. Using Argo CD to Deploy Helm Charts.
### 4.1. Create an Application in Argo CD.
Before the introduction of Argo CD, the developers were manually deploying any Helm chart changes to the cluster. Now, using an Application, you will declaratively tell Argo CD how to deploy the Helm charts.

Start by creating an Application to deploy the `guestbook` Helm Chart from the repo.

1. Navigate to the Argo CD UI, and click "NEW APP".

2. In the top right, click "EDIT AS YAML".

3. Paste the contents of `apps/guestbook.yaml` from your repo.
   
   This manifest describes an Application. 
    - The name of the Application is `guestbook`.
    - The source is your repo with the Helm charts.
    - The destination is the cluster connected by the agent.
    - The sync policy will automatically create the namespace.

4. Click "SAVE".
   
   At this point, the UI has translated the Application manifest into the corresponding fields in the wizard.

5. In the top left, click "CREATE".

   The new app pane will close and show the card for the Application you created. The status on the card will show "Missing" and "OutOfSync".

6. Click on the Application card titled `argocd/guestbook`.
   
   In this state, the Application resource tree shows the manifests generated from the source repo URL and path defined. You can click "APP DIFF" to see what manifests the Application rendered. Since auto-sync is disabled, the resources do not exist in the destination yet.

7. In the top bar, click "SYNC" then "SYNCHRONIZE" to instruct Argo CD to create the resources defined by the Application.

The resource tree will expand as the Deployment creates a ReplicaSet that makes a pod, and the Service creates an Endpoint and EndpointSlice. The Application will remain in the "Progressing" state until the pod for the deployment is running.

Afterwards, all the top-level resources (i.e., those rendered from the Application source) in the tree will show a green checkmark, indicating that they are synced (i.e., present in the cluster).

### 4.2. Syncing changes manually
An Application now manages the deployment of the `guestbook` Helm chart. So what happens when a developer wants to deploy a new image tag?

Well, instead of running `helm upgrade guestbook ./guestbook`, they will trigger a sync of the Application.

1. Navigate to your repo on Github, and open the file `guestbook/values.yaml`.

   `https://github.com/morey-tech/managed-argocd-lab/blob/2022-11-webinar/guestbook/values.yaml`

2. In the top right of the file, click the pencil icon to edit.

3. Update the `image.tag` to the `0.2` list.

4. Click "Commit changes...".

5. Add a commit message. For example `chore(guestbook): bump tag to 0.2`.

6. Click "Commit changes".

7. Switch to the Argo CD UI and go to the `argocd/guestbook` Application.

8. In the top right, click the "REFRESH" button to trigger Argo CD to check for any changes to the Application source and resources.

   The default sync interval is 3 minutes. Any changes made in Git may not apply for up to 3 minutes.

9. In the top bar, click "SYNC" then "SYNCHRONIZE" to instruct Argo CD to deploy the changes.

Due to the change in the repo, Argo CD will detect that the Application is out-of-sync. It will template the Helm chart (i.e., `helm template`) and patch the `guestbook` deployment with the new image tag, triggering a rolling update.

### 4.3. Enable auto-sync and self-heal for the guestbook Application.
Now that you are using an Application to describe how to deploy the Helm chart into the cluster, you can configure the sync policy to automatically apply changes—removing the need for developers to manually trigger a deployment for changes that already made it through the approval processes.

1. In the top menu, click "APP DETAILS".

2. Under the "SYNC POLICY" section, click "ENABLE AUTO-SYNC" and on the prompt, click "OK".

3. Below that, on the right of "SELF HEAL", click "ENABLE".

4. In the top right of the App Details pane, click the X to close it.

If the Application were out-of-sync, this would immediately trigger a sync. In this case, your Application is already in sync, so Argo CD made no changes.

### 4.4. Demonstrate Application auto-sync via Git.
With auto-sync enabled on the `guestbook` Application, changes made to the `main` branch in the repo will be applied automatically to the cluster. You will demonstrate this by updating the number of replicas for the `guestbook` deployment.

1. Navigate to your repo on Github, and open the file `guestbook/values.yaml`.

   `https://github.com/morey-tech/managed-argocd-lab/blob/2022-11-webinar/guestbook/values.yaml`

2. In the top right of the file, click the pencil icon to edit.

3. Update the `replicaCount` to the `2` list.

4. In the top right, click "Commit changes...".

5. Add a commit message. For example `chore(guestbook): scale to 2 replicas`.

6. In the bottom left, click "Commit changes".

7. Switch to the Argo CD UI and go to the `argocd/guestbook` Application.

8. In the top right, click the "REFRESH" button to trigger Argo CD to check for any changes to the Application source and resources.

You can view the details of the sync operation by, in the top menu, clicking "SYNC STATUS". Here it will display, what "REVISION" it was for, what triggered it (i.e., "INITIATED BY: automated sync policy"), and the result of the sync (i.e., what resources changed).

### 4.5. Demonstrate Application self-heal functionality.
In your organization, everyone has direct and privileged access to the cluster. Users may apply changes to the cluster outside of the repo due to the over-provisioned access. For example, applying a Helm chart change to the cluster without getting pushed to the repo first.

With self-heal enabled, Argo CD will reconcile any changes to the Application resources that deviate from the repo.

To demonstrate this:
1. From the `guestbook` Application page in the Argo CD UI:

2. Locate the `guestbook` deploy (i.e., Deployment) resource and click the three vertical dots on the right side of the card.

3. Then click "Delete".

4. Enter the deployment name `guestbook` and click "OK".

Almost as quickly as you delete it, Argo CD will detect that the deploy resource is missing from the Application. It will briefly display the yellow circle with a white arrow to indicate that the resource is out-of-sync. Then automatically recreate it, bringing the Application back to a healthy status.

## 5. Managing Argo CD Applications declaratively.
### 5.1. Create an App of Apps.
One of the benefits of using Argo CD is that you are now codifying the deployment process for the Helm charts in the Application spec.

Earlier in the lab, you created the `guestbook` Application imperatively, using the UI. But what if you want to manage the Application manifests declaratively too? This is where [the App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern) comes in.

1. Navigate to the Applications dashboard in the Argo CD UI, and click "NEW APP".

2. In the top right, click "EDIT AS YAML".

3. Paste the contents of `app-of-apps.yaml` (in the repo's root).
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: applications
      namespace: argocd
    spec:
      destination:
        name: in-cluster
      project: default
      source:
        path: apps
        repoURL: https://github.com/<github-username>/managed-argocd-lab # Update to your repo URL.
        targetRevision: HEAD
    ```

    This Application will watch the `apps/` directory in your repo which contains Application manifests for the `guestbook` and `portal` Helm charts.

4. Update `<github-username>` in `spec.source.repoURL` to match your GitHub username.

5. Click "SAVE".

6. Then, in the top left, click "CREATE".

7. Click on the Application card titled `argocd/applications`.

    At this point, the Application will be out-of-sync. The diff will show the addition of the `argocd.argoproj.io/tracking-id` label to the existing `guestbook` Application, which indicates that the "App of Apps now manages it".

    ```diff
      kind: Application
      metadata:
    ++  annotations:
    ++    argocd.argoproj.io/tracking-id: 'applications:argoproj.io/Application:argocd/guestbook'
        generation: 44
        labels:
          ...
          path: guestbook
          repoURL: 'https://github.com/<github-username>/managed-argocd-lab'
        syncPolicy:
          automated: {}
    ```

    Along with a new Application for the `portal` Helm chart.

8. To apply the changes, in the top bar, click "SYNC" then "SYNCHRONIZE".

From this Application, you can see all of the other Applications managed by it in the resource tree. Each child Application resource has a link to its view on the resource card.

## 6. Review.
You have reached the end of the lab. You have an Argo CD instance deploying Helm charts from your repo into your cluster.

The developers no longer manually deploy Helm chart changes. The Application spec describes the process for deploying the Helm charts declaratively. There's no longer a need for direct access to the cluster, removing the primary source of configuration drift.
