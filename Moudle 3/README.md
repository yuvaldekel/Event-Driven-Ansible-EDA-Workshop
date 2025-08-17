## Module 3: Configuring EDA and AAP

### 3.1 Exercise: Configuring Credentials in EDA and AAP

1.  **In the EDA UI (within AAP):**
    * **Image Registry Credential:**
        * Name: `Quay Credential`
        * Type: `Container Registry`
        * URL: `registry.redhat.io` #from which we will pull the DE image
        * usename: ??????????
        * token: ?????????
    * **AAP Controller Credential:**
        * Name: `AAP Controller Credential`
        * Type: `Controller`
        * Host: `https://<URL of the AAP controller route>/api/controller/`
        * User: `admin`.
        * Get Password: `oc get secret <instance-name>-controller-admin-password -o jsonpath='{.data.password}' | base64 --decode`
    * **Git Repo Credential:**
        * Name: `GitLab/GitHub Credential`
        * Type: `Source Control`.

2.  **In the AAP UI:**
    * **OpenShift Cluster Credential:**
        * Create a new Credential Type for OpenShift (`openshift_api_url`, `openshift_token`).
        * Create a new credential of this type for your cluster(s).

### 3.2 Exercise: Creating the Project in EDA

1.  Go to `Projects` -> `Create project`.
2.  Name: `OpenShift Alerting Project`.
3.  Point to your forked Git repository URL and select your Git credential.
4.  Click `Save` and wait for the project to sync.

### 3.3 Exercise: Referencing the Decision Environment

Before creating the Rulebook Activation, you need to configure the Decision Environment that will execute your Event-Driven Ansible rulebooks.

1.  **Navigate to Decision Environments in AAP:**
    * In the AAP UI, go to **Administration** → **Execution Environments**
    * Click **Add** to create a new execution environment

2.  **Configure the Red Hat Decision Environment:**
    * **Name**: `EDA Decision Environment`
    * **Image**: `registry.redhat.io/ansible-automation-platform-25/de-supported-rhel9`
    * **Description**: `Red Hat supported Decision Environment for Event-Driven Ansible`
    * **Pull Policy**: `Always` (recommended for production)
    * **Credential**: Reference the `Image Registry Credential` we've generated in step `3.1`

**Important**: This decision environment will be used in the next section when creating your Rulebook Activation.

### 3.4 Exercise: Creating the Rulebook Activation

1.  Go to `Rulebook Activations` -> `Create rulebook activation`.
2.  Name: `alertmanager-listener`
3.  Project: `OpenShift Alerting Project`.
4.  Rulebook: `extensions/eda/rulebooks/rulebook.yml`.
5.  Decision Environment: Select `EDA Decision Environment` (the Red Hat pre-built image configured in the previous section).
6.  Click `Create rulebook activation`.
7.  **Apply the Route:**
    * Now that the activation has created the `alertmanager-listener` service, apply the route manifest you created earlier.
        ```bash
        # Make sure you are in your local workshop repo directory
        oc apply -f extensions/eda/k8s-objects/eda-route.yml
        ```
8.  **Get the Webhook URL:**
    * Find the exposed URL for the webhook. This is the endpoint you'll give to Alertmanager.
        ```bash
        oc get route alertmanager-listener -n <aap-namespace> -o jsonpath='{.spec.host}'
        ```
    * Your full webhook URL will be `https://<route-hostname-from-above>/api/eda/v1/alerts`. Note that we use **https** now because of the edge termination.

