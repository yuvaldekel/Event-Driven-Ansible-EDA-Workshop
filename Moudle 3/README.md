## Module 3 - Part 1: Configuring EDA

### 3.1 Exercise: Configuring Credentials in EDA

1.  **In the EDA UI (within AAP), go to Automation Desicions -> Infrastrucutre -> Credentials, and create the following:**
    * **Image Registry Credential:**
        * Name: `Red Hat Registry Credential`
        * Type: `Container Registry`
        * URL: `registry.redhat.io` #from which we will pull the DE image
        * Usename: No need, public registry
        * Token: No need, public registry
    * **AAP Controller Credential:**
        * Name: `AAP Controller Credential`
        * Type: `Controller`
        * Host: `https://<URL of the AAP controller route>/api/controller/`
        * User: `admin`.
        * Get Password: `oc get secret <instance-name>-controller-admin-password -o jsonpath='{.data.password}' | base64 --decode`
    * **Git Repo Credential:**
        * Name: `GitLab/GitHub Credential`
        * Type: `Source Control`
        * Username: ?????????
        * Token: ??????????????
    
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

## Module 3 - Part 2: Configuring AAP

### 3.5 Exercise: Configuring Credentials in AAP
**Git Repo Credential:**
  * Name: `GitLab/GitHub Credential`
  * Type: `Source Control`
  * Username: ?????????
  * Token: ??????????????

**OpenShift Cluster Credential:**
1. Create a new Credential Type for OpenShift (`openshift_api_url`, `openshift_token`):
   * Navigate to `Authentication Execution` -> `Infrastructure` -> `Credential Types` -> `Create New`
   * Name: `OCP Login`

   * Input configuration:
      ```yaml
      fields:
      - id: dev_ocp_api
        type: string
        label: DEV OCP API Endpoint
      - id: dev_ocp_token
        type: string
        label: DEV OCP Token of ServiceAccount aap-eda-sa in openshift-monitoring namespace
      - id: prd_ocp_api
        type: string
        label: PRD OCP API Endpoint
      - id: prd_ocp_token
        type: string
        label: PRD OCP Token of ServiceAccount aap-eda-sa in openshift-monitoring namespace
      ```

    * Injector configuration:
      ```yaml
      env:
        DEV_OCP_API: '{{ dev_ocp_api }}'
        DEV_OCP_AUTH_TOKEN: '{{ dev_ocp_token }}'
        PRD_OCP_API: '{{ prd_ocp_api }}'
        PRD_OCP_AUTH_TOKEN: '{{ prd_ocp_token }}'
      ```

    2. Create a new credential of this type for your cluster(s):
    * Navigate to `Authentication Execution` -> `Infrastructure` -> `Credential` -> `Create New`
    * Name: `OCP Login Creds`
    * Credential: `OCP Login`
    * Type Details:
      * DEV OCP API Endpoint: `<https://dev_URL:6443>`
      * DEV OCP Token of ServiceAccount aap-eda-sa in openshift-monitoring namespace: `<PRINT SAVED TOKEN FROM Module 2 HERE>`
      * PRD OCP API Endpoint: `<https://prd_URL:6443>`
      * PRD OCP Token of ServiceAccount aap-eda-sa in openshift-monitoring namespace: `<PRINT SAVED TOKEN FROM Module 2 HERE>`


### 3.6 Exercise: Creating the Project in AAP

1.  Go to `Authentication Execution` -> `Projects` -> `Create project`.
2.  Name: `OpenShift Automations`.
3.  Point to your forked Git repository URL and select your Git credential.
4.  Click `Save` and wait for the project to sync.

### 3.6 Exercise: Creating a project template in AAP

1.  Go to `Authentication Execution` -> 
2.  Name: 
3.  Point to your forked Git repository URL and select your Git credential.
4.  Point to the OCP credential we created earlier
4.  Click `Save`