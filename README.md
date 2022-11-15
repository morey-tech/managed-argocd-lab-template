# Creating a fully-managed Kubernetes GitOps platform with Argo CD

In this hour-long hands-on lab, we will set up a repo to contain the environment configuration, create a Kubernetes cluster, and provision an Argo CD instance. Then use Argo CD to deploy Applications from the repo to the cluster. Finally, we will demonstrate the auto-sync and auto-heal functionality of the Applications.

Ultimately, you will have a Kubernetes cluster, with Applications deployed using an Argo CD control plane.

Use these links to learn what is: [Kubernetes](https://youtu.be/4ht22ReBjno?t=164), [GitOps](https://opengitops.dev/), [Argo CD](https://argo-cd.readthedocs.io/en/stable/#what-is-argo-cd), [Akuity](https://akuity.io/).

- [Lab Scenario](#lab-scenario)
- [Pre-requisetes](#pre-requisetes)
- [Setting up your environment.](#setting-up-your-environment)
  - [1. Create repository from template.](#1-create-repository-from-template)
  - [2. Create a Kubernetes cluster using `kind`.](#2-create-a-kubernetes-cluster-using-kind)
  - [3. Launch an Argo CD instance.](#3-launch-an-argo-cd-instance)
  - [4. Deploy an agent to the cluster.](#4-deploy-an-agent-to-the-cluster)
- [Using Argo CD to Deploy Helm Charts.](#using-argo-cd-to-deploy-helm-charts)
  - [5. Create an Application in Argo CD.](#5-create-an-application-in-argo-cd)
  - [6. Enable auto-sync and auto-heal for the guestbook Application.](#6-enable-auto-sync-and-auto-heal-for-the-guestbook-application)
  - [7. Demonstrate Application auto-sync via Git.](#7-demonstrate-application-auto-sync-via-git)
  - [8. Demonstrate Application auto-heal functionality.](#8-demonstrate-application-auto-heal-functionality)
- [Managing Argo CD Applications declaritively.](#managing-argo-cd-applications-declaritively)
  - [9. Create an App of Apps.](#9-create-an-app-of-apps)

## Lab Scenario
You are a Kubernetes cluster administrator who works in an organization, where containerized applications are deployed using Helm charts.

The charts are managed in a GitHub repo by developers. They connect directly to the cluster to deploy changes from the `main` branch using `helm`. Which requires privileged access.

To prepare for the lab scenario, you can think of what to name your *organization* and *environment*. For example, your *organization* could be your Github username and your *environment* “laptop”.

- The *organization* name will be used for creating an organization, while setting up your account on the Akuity Platform.
- The *environment* name will be used for the cluster and the App of Apps Application in Argo CD.

Anywhere you see text in the format `<...>`, this indicates that you need to replace it with the value relevant to your scenario. Using my scenario for example, in the repo `https://github.com/<github-username>/managed-argocd-lab`, I would replace `<github-username>` with `morey-tech`.

## Pre-requisetes
The lab requires that you have:
- a [GitHub](https://github.com/) Account - This will be used to create the repo for the environment configuration and for creating an account on the [Akuity Platform](https://akuity.io/akuity-platform/).
- the Kubernetes command-line tool, [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) - Will be used for interacting with the cluster directly.
- a Kubernetes cluster with Cluster Admin access (to create namespaces and cluster roles) or;
- [Docker Desktop](https://duckduckgo.com/?q=docker+desktop+install) and [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed on your local machine.
- [egress traffic](https://www.webopedia.com/definitions/egress-traffic/) allowed from the Kubernetes cluster to the internet.

The lab was built and tested using the following tool and component versions:
- Argo CD: v2.5.1
- Docker Desktop: 4.13.1
- Kubernertes: v1.25.2
- kind: v0.16.0

## Setting up your environment.
### 1. Create repository from template.
You will use a template repository containing the application Helm charts for the scenario.

Start by creating a repository on your Github account from the template. 
1. Click [this link](https://github.com/morey-tech/managed-argocd-lab/generate) or click "Use this template" from the repo main page.
2. Ensure the desired "Owner" is selected (e.g., your personal account and not an org).
3. Enter a `managed-argocd-lab` for the "Repository name". 
4. Then click "Create repository from template".

Next, there are a couple of files to update in the to reflect your repository.
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
++    name: laptop
```

### 2. Create a Kubernetes cluster using `kind`.
If you brought your own Kubernetes cluster to the lab, you can skip this section.

Otherwise, you will be creating one on your local mahcine using `kind`.

1. Create a cluster using `kind` and specify your *environment* name.
   ```
   kind create cluster --name <environment-name>
   ```

   You should see the follow output:
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

2. Check that the cluster is working by running `kubectl get nodes`.
   ```
   % kubectl get nodes
   NAME                               STATUS   ROLES           AGE   VERSION
   <environment-name>-control-plane   Ready    control-plane   74s   v1.25.2
   ```

   This will demonstrate that `kubectl` can connect to the cluster and query the API server. The node should be in the "Ready" status.

### 3. Launch an Argo CD instance.
This is were the "fully-managed" part comes in. The Akuity Platform provides Argo CD as a service. By spinning up an instance on Akuity, you'll have an external Argo CD control plane that can manage clusters in private networks. 

Just like how GitHub is hosting the manifests for your apps, the Akuity Platform will host, not only Argo CD, but all of the Application, ApplicationSet, and AppProject definitions. This removes the need for a Kubernetes cluster to host Argo CD, so that it can manage other clusters.

1. Create an account on the [Akuity Platform](https://akuity.io/signup).
2. To log in with GitHub SSO, click "Continute with GitHub".
   1. You can also use Google SSO or a email and password combo. For the sake of the lab, I'm assuming you will be using GitHub.
3. Click "Authorize akuityio".
4. Create an organization by clicking "create or join" in the information bulletin.
5. In the top right, click "New organization".
6. Enter your "Organization Name" and click "Create"

7. Near the top of the sidebar, click "Argo CD".
8. In the top right, click "Create".
9. Set the "Instance Name" to `cluster-manager`.
10. Under the "Version" section, click the option correisponding to `v2.5`.
11. Click "Create".

    At this point, your Argo CD instance will begin intializing. This typically takes under 2 minutes.

    To get the instance ready for the rest of the lab, there are a couple of steps left.

12. In the dashboard for the Argo CD instance, click "Settings".
13. In the "General" section, find "Declarative Management" and enable it by clicking the toggle.
    
    This enables using the Akuity Platform with the App of Apps pattern or ApplicationSets. We will demonstrate this later in the lab.

14. In the top right, click "Save".
15. On the inner sidebar, under "Security & Access", click "System Accounts".
16. Enable the "Admin Account" by clicking the toggle, and clicking "Confirm" on the prompt.
17. Then for the `admin` user, click "Set Password".
18. To get the password, click "Copy".
   - If the copy option does not appear, click "Regenerate Password" and then "Copy". Note, this will invalidate any other sessions for the `admin` user.
15. In the bottom right of the Set password prompt, hit "Close"
16. In the top, next to the Argo CD instance name and status, click the instance URL (e.g., `<uuid>.cd.akuity.cloud`). This will open the Argo CD login page.
17.  Enter the username `admin` and the password copied previously.

You will be presented with the Argo CD Applications dashboard.

### 4. Deploy an agent to the cluster.
Now that you have an Argo CD instance running and assecible, you need to connect your cluster to it. With the Akuity Platform, you will deploy an agent to the cluster enabling it to connect it to the control plane.

1. Back on the Akuity Platform, in the top left of the dashboard for the Argo CD instance, click "Clusters".
2. In the top right, click "Connect a cluster".
3. Enter your *environment* name as the "Cluster Name".
4. In the bottom right, Click "Connect Cluster".
5. To get the agent install command, click "Copy to Clipboard". Then, in the bottom right, "Done".
6. Open your terminal and ensure that are targeting the correct cluster with `kubectl config current-context`. 
    
    If you are following along using `kind`, you should see:
    ```
    % kubectl config current-context
    kind-<env-name>
    ```
    - This output assumes you are using `kind`.
7. Paste and run the command against the cluster.
   1. This will create the `akuity` namespace and deploy the resources for the Akuity Agent.
8. Check the pods in the `akuity` namespace. Wait for the `Running` status on all of the pods (apprx. 1 minute).
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
   - Re-run the `get pods` command to check for updates on the pod statuses.
9.  Back on the Clusters dashboard, confirm that the cluster shows a green heart before the name, indicating a healthy status.

## Using Argo CD to Deploy Helm Charts.
### 5. Create an Application in Argo CD.
An Application declaritively tells Argo CD what Kubernetes recoures to deploy. In this section, you will create one to deploy the `guestbook` Helm Chart, from your GitOps repo, to your cluster.

1. Navigate back to the Argo CD UI, and click "NEW APP".
2. In the top right, click "EDIT AS YAML".
3. Paste the contents of `apps/guestbook.yaml` from your repo.
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: guestbook
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: 'https://github.com/<github-username>/managed-argocd-lab' # Update to match your fork.
        path: guestbook
        targetRevision: HEAD
      destination:
        namespace: guestbook
        name: <environment-name> # Update this value.
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
    ```

    This manifest describes an Application. 
    - The name of the Application is `guestbook`.
    - The source is your repo with the Helm charts.
    - The destination is the cluster you deployed the agent to.
    - The sync policy will automatically create the namespace.

4. Replace `<github-username>` in `spec.source.repoURL` to match your Github username.
5. Update `<environment-name>` in `spec.destination.name` to match your environment name (i.e., cluster name).
6. Click "SAVE".
   
   At this point, the UI has translated the Application manifest into the corriesponding fields in the wizard.

7. In the top left, click "CREATE".

   This will close the new app pane and show the card for the Application you created. The status on the card will show "Missing" and "OutOfSync".

8. Click on the Application card titled `argocd/guestbook`.
   
   In this state, the Application resource tree shows what was generated from the source repo URL and path defined. You can click "APP DIFF" to see what manifests the Application rendered. Since auto-sync is disable, the resources does not exist in the destination yet.

9.  In the top bar, click "SYNC" then "SYNCHRONIZE". This will instruct Argo CD to create the resources defined by the Application.

The tree will expand as the Deployment creates a ReplicaSet that then creates a pod, and the Service creates an Endpoint and EndpointSlice. The Application will remain in the "Progressing" state until the pod for the deployment is running.

Afterwards, all the top-level resources (i.e., those render from the Application source) in the tree will show a green checkmark. Indicating that they are synced (i.e., present in the cluster).

### 6. Enable auto-sync and auto-heal for the guestbook Application.
You can automate the deployment of Helm chart changes, by enabling the automated sync policy. This will cause any change to the Application source in the repo, or the Application itself, to be applied without intervention.

1.  In the top menu, click "APP DETAILS".
2.  Under the "SYNC POLICY" section, click "ENABLE AUTO-SYNC" and on the prompt, click "OK".
3.  Below that, on the right of "SELF HEAL", click "ENABLE".
4.  In the top right of the App Details pane, click the X to close it.

If the Application was out-of-sync, this would immediately trigger a sync. In this case, your Application is already in sync, so no changes are made.

The default sync interval is 3 minutes. Meaning, any changes made in Git may not apply for up to 3 minutes.

### 7. Demonstrate Application auto-sync via Git.
With auto-sync enabled on the `guestbook` Application, you can now make a change in Git to have it applied automatically in your cluster. You will demonstrate this by updating the image tag for the `guestbook` Application.

1. From your repo in Github, navigate to the `guestbook/values.yaml`.

   `https://github.com/<github-username>/managed-argocd-lab`

2. In the top right of the file, click the pencil icon to edit.
3. Update the `image.tag` to the `0.2` list.
4. In the top right, click "Commit changes...".
5. Add a commit message. For example `chore: bump tag to 0.2 for guestbook`.
6. In the bottom left, click "Commit changes".
7. Switch back to the Argo CD UI and go to the `argocd/guestbook` Application.
8.  In the top right, click the "REFRESH" button.
   1. This will trigger Argo CD to check for any changes to the Application source and resources. This would happen automatically on a 3 minute (default) interval.

Due to the change in the repo, Argo CD will detect that the Application is out-of-sync. It will template the Helm chart, do a three-way diff, and patch the `guestbook` deployment with the new image tag, triggering a rolling update.

You can view the details of the sync operation by, in the top menu, clicking "SYNC STATUS". Here it will display, what "REVISION" it was for, what triggered it (i.e., "INITATED BY: automated sync policy"), and the result of the sync (i.e., what resources changed).

### 8. Demonstrate Application auto-heal functionality.
The auto-heal functionality will ensure that, any changes made in the cluster to the Application's resources, that deviate from the desired state in the GitOps repo, will be reconciled.

To demonstrate this:
1. From the `guestbook` Application page in the Argo CD UI:
2. Locate the `guestbook` deploy (i.e., Deployment) resource and, on the right side of the card, click the three vertical dots.
3. Then click "Delete".
4. Enter the Application name `guestbook` and click "OK".

Almost as quickly as the it's deleted, Argo CD will detect that the deploy resource is missing from the Application. It will breifly display the yellow circle with a white arrow, to indicate the resource is out-of-sync. Then automatically recreate it, bringing the Application back to a healthy status.

## Managing Argo CD Applications declaritively.
### 9. Create an App of Apps.
Earlier in the lab, you created the `guestbook` Application using the UI (i.e., imparatively). What if you wanted the Application manifests to be managed declaratively too? This is where [the App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern) (or ApplicationSets) comes in. You will create a new Application that will manage the Application manifests for the apps.

1. Navigate back to Applications dashboard in the Argo CD UI, and click "NEW APP".
2. In the top right, click "EDIT AS YAML".
3. Paste the contents of `app-of-apps.yaml` (in the root of the repo).
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: <environment-name> # Update to your cluster name.
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

4. Update `<environment-name>` in `metadata.name` to match your environment name (i.e., cluster name).
5. Update `<github-username>` in `spec.source.repoURL` to match your Github username.
6. Click "SAVE".
7. Then, in the top left, click "CREATE".
8. Click on the Application card titled `argocd/<environment-name>`.

    At this point, the Application will be out-of-sync. The diff will show the addition of the `argocd.argoproj.io/tracking-id` label to the existing `guestbook` Application. This indicates that it's now managed by the "App of Apps".

    ```diff
      kind: Application
      metadata:
    ++  annotations:
    ++    argocd.argoproj.io/tracking-id: '<environment-name>:argoproj.io/Application:argocd/guestbook'
        generation: 44
        labels:
          ...
          path: guestbook
          repoURL: 'https://github.com/<environment-name>/managed-argocd-lab'
        syncPolicy:
          automated: {}
    ```

    Along with a new Application for the `portal` Helm chart.

9.  To apply the changes, in the top bar, click "SYNC" then "SYNCHRONIZE".