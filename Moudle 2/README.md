## Module 2: Lab Setup and Configuration

### 2.1 Exercise: Git Repository Structure

A well-organized Git repository is crucial for a scalable and maintainable automation project. EDA and AAP expect a certain structure to locate all the necessary components like rulebooks, playbooks, and variables.

1.  **Create the required directories:**
    * In your cloned `eda-openshift-workshop` directory, run the following commands to create the structure:
        ```bash
        mkdir -p extensions/eda/k8s-objects
        mkdir -p extensions/eda/playbooks
        mkdir -p extensions/eda/rulebooks
        mkdir -p templates
        mkdir -p vars
        ```

2.  **Create the environment variable files:**
    * Using your text editor, create a new file at `vars/prd.yml` and add the following content, replacing the values with your **production** cluster details:
        ```yaml
        ---
        openshift_api: "https://api.your-prd-cluster.com:6443"
        openshift_token: "sha256~your-prd-token"
        ```
    * Create another file at `vars/dev.yml` with your **development** cluster details:
        ```yaml
        ---
        openshift_api: "https://api.your-dev-cluster.com:6443"
        openshift_token: "sha256~your-dev-token"
        ```
    * **Explanation**: The remediation playbook will dynamically load one of these files based on an `env` label in the incoming alert.
    * **Security Best Practice:** Storing plain-text tokens directly in a Git repository is a significant security risk. We are doing this now for simplicity. In a later module, we will replace this with a secure reference to an Ansible Automation Platform credential.

3.  **How to Generate the `openshift_token`**

    To get the token, you need to create a **Service Account** on each target OpenShift cluster (`dev` and `prd`). The following steps require a **cluster-admin** user.

    **On each target cluster, perform the following steps:**

    1.  **Log in to the Target OpenShift Cluster** with a cluster-admin user.
    2.  **Create a Service Account in a Restricted Namespace**:
        ```bash
        oc create sa aap-eda-sa -n openshift-monitoring
        ```
    3.  **Grant Cluster-Admin Permissions**:
        ```bash
        oc adm policy add-cluster-role-to-user cluster-admin -z aap-eda-sa -n openshift-monitoring
        ```
    4.  **Generate a Token for the Service Account**:
        ```bash
        oc create token aap-eda-sa -n openshift-monitoring
        ```
    The output is the `sha256~...` token. Copy this and paste it into the corresponding `vars/dev.yml` or `vars/prd.yml` file.

4.  **Create the OpenShift Route manifest:**
    * Create a new file at `extensions/eda/k8s-objects/eda-route.yml` and add the following content. **Replace `<aap-namespace>` with the actual namespace where your AAP instance is installed.**
        ```yaml
        apiVersion: route.openshift.io/v1
        kind: Route
        metadata:
          name: eda-webhook-route
          namespace: <aap-namespace> # <-- IMPORTANT: Set this to your AAP namespace
        spec:
          to:
            kind: Service
            name: alertmanager-listener
          port:
            targetPort: 5001
          tls:
            termination: edge
            insecureEdgeTerminationPolicy: Redirect
        ```
    * **Explanation**: This Route manifest makes the EDA listener accessible to the OpenShift Alertmanager.
        * `namespace`: The Route must be in the same namespace as the Service it exposes. Since the Rulebook Activation service will be created in your AAP namespace, the Route must be there as well.
        * `name: alertmanager-listener`: This must **exactly match** the name of the Rulebook Activation you will create later, as EDA uses that name to create the corresponding Service.
        * `tls: { termination: edge, insecureEdgeTerminationPolicy: Redirect }`: This is a critical security configuration. It tells the OpenShift router to terminate TLS (HTTPS) at the edge of the cluster. Any external, unencrypted HTTP traffic will be automatically redirected to HTTPS. This ensures that the alert data sent from Alertmanager is encrypted as it travels over the network to your cluster, protecting it from snooping.

5.  **Create the Rebooter Pod Template:**
    * Create a new file at `templates/reboot_node_pod.yaml.j2` and add the following content:
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: rebooter-{{ problematic_node }}
          namespace: openshift-monitoring
        spec:
          serviceAccountName: aap-eda-sa
          restartPolicy: Never
          nodeName: "{{ problematic_node }}"
          hostPID: true
          hostNetwork: true
          containers:
            - name: "rebooter"
              image: "registry.redhat.io/rhel8/support-tools"
              command: ["/bin/bash", "-c"]
              args:
                - |
                  chroot /host /bin/bash -c "reboot"
              securityContext:
                privileged: true
        ```

6.  **Commit and Push Your Initial Structure:**
    * Now that you've created the initial files and directories, save them to your Git repository. This is a crucial step, as AAP will pull this code later.
        ```bash
        git add .
        git commit -m "Initial workshop structure and config files"
        git push
        ```

### 2.2 Exercise: Creating a Lean Decision Environment

1.  **Create `requirements.yml` and `execution-environment.yml` files** in the root of your Git repo with the content below.
    * `requirements.yml`:
        ```yaml
        ---
        collections:
          - ansible.eda
        ```
    * `execution-environment.yml`:
        ```yaml
        ---
        version: 1
        build_arg_defaults:
          EE_BASE_IMAGE: 'registry.redhat.io/ansible-automation-platform-2.4/de-minimal-rhel8:latest'
        dependencies:
          galaxy: requirements.yml
        ```

2.  **Build and Push the DE Image:**
    * Use `ansible-builder` to build the image:
        ```bash
        ansible-builder build -f execution-environment.yml -t quay.io/your-username/eda-de:latest
        ```
    * Log in to your image registry:
        ```bash
        podman login quay.io
        ```
    * Push the image:
        ```bash
        podman push quay.io/your-username/eda-de:latest
        ```

### 2.3 Exercise: Writing the Rulebook

1.  **Create a file named `rulebook.yml`** inside the `extensions/eda/rulebooks/` directory and add the following content:
    ```yaml
    ---
    - name: Listen for OpenShift alerts
      hosts: all
      sources:
        - ansible.eda.alertmanager:
            host: 0.0.0.0
            port: 5001
      rules:
        - name: "A node health check has failed"
          condition: >
            event.alert.status == "firing" and
            (event.alert.labels.alertname == "NodeHealthCheckFailed" or event.alert.labels.alertname == "NFSStale") and
            event.alert.labels.env is defined
          actions:
            - run_job_template:
                name: "Drain and Reboot Unhealthy Node"
                organization: "Default"
                job_args:
                  extra_vars:
                    payload: "{{ event.alert }}"
    ```
    * **Explanation:** This rulebook is a central entry point for all alerts. The `condition` block acts as a filter, ensuring only specific node health alerts trigger the `run_job_template` action.

### 2.4 Exercise: Writing the Remediation Playbook

1.  **Create a file named `drain_and_reboot_node.yml`** inside the `extensions/eda/playbooks/` directory and add the following content:
    ```yaml
    ---
    - name: "Playbook reacting to alerts nfs-stale OR node-health-check"
      hosts: localhost
      gather_facts: false
      vars_prompt:
        - name: payload
          prompt: ""
          private: false
   
      pre_tasks:
        - name: Set fact node_name based on existing label from alert
          set_fact:
            problematic_node: "{{ payload.labels[item] }}"
          loop:
            - node
            - instance
          when: payload.labels is defined and item in payload.labels
   
        - name: "Include vars file for the target environment"
          include_vars: ../../vars/{{ payload.labels.env }}.yml
   
      tasks:
        - name: "Log in to the correct OpenShift cluster"
          shell: "oc login --insecure-skip-tls-verify=true --token={{ openshift_token }} {{ openshift_api }}"
   
        - name: "Validation: Don’t reboot node if you have more then 1 NotReady nodes or 3 alert simultaniously"
          shell: |
            numOfAlertsSimultaniuously=`oc -n openshift-monitoring exec -it $(oc -n openshift-monitoring get pod -l alertmanager=main -o jsonpath='{.items[0].metadata.name}') -- curl -s http://localhost:9093/api/v2/alerts | jq '.[] | select(.labels.alertname=="detect-nfs-stale" or .labels.alertname=="node-health-check") | .fingerprint' | sort -u | wc -l` 
            numOfNotReadyNodes=`oc get nodes | grep -E 'NotReady|SchedulingDisabled' | wc -l`
            if [ "$numOfAlertsSimultaniuously" -gt 3 ] || [ "$numOfNotReadyNodes" -gt 1 ] ; then
              exit 1;
            else
              exit 0;
            fi
          register: validations_status
          until: validations_status.rc == 0
          retries: 10
          delay: 600
   
        - name: "Drain node of its pods"
          shell: "oc adm drain {{ problematic_node }} --force --delete-emptydir-data --ignore-daemonsets"
          register: drain_status
   
        - name: "Create rebooter pod manifest from template"
          template:
            src: ../../templates/reboot_node_pod.yaml.j2
            dest: "/tmp/reboot_node_pod-{{ problematic_node }}.yaml"
   
        - name: "Apply the rebooter pod manifest to trigger the node reboot"
          shell: "oc apply -f /tmp/reboot_node_pod-{{ problematic_node }}.yaml"
    ```

    * **Task Explanations:**
        * **`vars_prompt`:** We need this part so the playbook define the payload variable we transferred to it from the rulebook activation.
        * **`pre_tasks`:** These tasks run before the main remediation logic to prepare the environment.
            * **`Set fact node_name...`:** This task extracts the name of the problematic node from the alert data. It checks for both `node` and `instance` labels to make the playbook more robust.
            * **`Include vars file...`:** This is the core of the multi-cluster strategy. It uses the `env` label from the alert (e.g., `prd`) to dynamically load the correct variables file (`vars/prd.yml`), which contains the API server URL and token for the target cluster.
        * **`tasks`:** This is the main execution block.
            * **`Log in to the correct OpenShift cluster`:** Uses the variables loaded in the pre-tasks to authenticate against the correct remote OpenShift cluster.
            * **`Validation...`:** This is a critical safety check. Before performing a disruptive action, it runs a shell command inside the cluster to check two conditions: (1) Are there already more than 3 node-health alerts firing? and (2) Are there already more than 1 `NotReady` nodes? If either is true, the playbook will pause for 10 minutes and retry, preventing a cascading failure if the cluster is already unstable.
            * **`Drain node of its pods`:** Once the safety checks pass, this task gracefully evicts all pods from the unhealthy node using `oc adm drain`, ensuring workloads are moved to healthy nodes before the reboot.
            * **`Create rebooter pod manifest...`:** Uses the Jinja2 template you created earlier (`reboot_node_pod.yaml.j2`) to generate a temporary pod manifest file.
            * **`Apply the rebooter pod manifest...`:** This final task applies the manifest, which creates the privileged pod on the target node, triggering the reboot and completing the remediation.

2.  **Commit and Push the Automation Code:**
    * Now that the core automation logic (rulebook and playbook) is written, push it to your repository so AAP can find it.
        ```bash
        git add .
        git commit -m "Add EDA rulebook and remediation playbook"
        git push
        ```
