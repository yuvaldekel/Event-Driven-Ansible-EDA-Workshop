## Module 2: Lab Setup and Configuration

### 2.0 Exercise: Fork and Clone the Workshop Repository

Before we begin setting up our Event-Driven Ansible lab environment, you'll need to fork and clone this repository to your own GitHub account. This will give you a personal copy of the workshop materials that you can modify and push changes to.

1.  **Fork the Repository:**
    * Navigate to the original workshop repository on GitHub: [Event-Driven-Ansible-EDA-Workshop](https://github.com/your-org/Event-Driven-Ansible-EDA-Workshop)
    * Click the **"Fork"** button in the top-right corner of the repository page
    * This creates a copy of the repository under your GitHub account

2.  **Clone Your Forked Repository:**
    * Copy the clone URL from your forked repository (use HTTPS or SSH based on your preference)
    * Open a terminal on your local machine and run:
        ```bash
        git clone https://github.com/YOUR-USERNAME/Event-Driven-Ansible-EDA-Workshop.git
        ```
    * Replace `YOUR-USERNAME` with your actual GitHub username

3.  **Navigate to the Repository Directory:**
    ```bash
    cd Event-Driven-Ansible-EDA-Workshop
    ```

4.  **Verify the Repository Structure:**
    * List the contents to ensure you have all the module directories:
        ```bash
        ls -la
        ```
    * You should see directories for all workshop modules (Module 0, Module 1, Module 2, etc.)

5.  **Configure Git (if not already done):**
    * Set your Git username and email if you haven't already:
        ```bash
        git config --global user.name "Your Name"
        git config --global user.email "your.email@example.com"
        ```

**Why Fork and Clone?**
* **Personal Workspace**: You'll have your own copy to experiment with and modify
* **Version Control**: All your changes will be tracked and can be pushed to your repository
* **AAP Integration**: Later in the workshop, Ansible Automation Platform will pull your code from your forked repository
* **Collaboration**: You can share your customized setup with others or contribute back to the original project

**Important Note**: Make sure to use your forked repository URL when configuring Ansible Automation Platform in later modules, as AAP will need to pull the rulebooks and playbooks you'll be creating and modifying.

### 2.1 Exercise: Git Repository Structure

A well-organized Git repository is crucial for a scalable and maintainable automation project. EDA and AAP expect a certain structure to locate all the necessary components like rulebooks, playbooks, and variables.

1.  **Create the required directories:**
    * In your cloned `Event-Driven-Ansible-EDA-Workshop` directory, run the following commands to create the structure:
        ```bash
        mkdir -p extensions/eda/k8s-objects
        mkdir -p extensions/eda/playbooks
        mkdir -p extensions/eda/playbooks/{templates,vars}
        mkdir -p extensions/eda/rulebooks
        ```

2.  **Create the environment variable files:**
    * Using your text editor, create a new file at `extensions/eda/playbooks/vars/prd.yml` and add the following content, replacing the values with your **production** cluster details:
        ```yaml
        ---
        openshift_api: "{{ lookup('env', 'PRD_OCP_API') }}"
        openshift_token: "{{ lookup('env', 'PRD_OCP_AUTH_TOKEN') }}"
        ```
    * Create another file at `extensions/eda/playbooks/vars/dev.yml` with your **development** cluster details:
        ```yaml
        ---
        openshift_api: "{{ lookup('env', 'DEV_OCP_API') }}"
        openshift_token: "{{ lookup('env', 'DEV_OCP_AUTH_TOKEN') }}"
        ```
    * **Explanation**: The remediation playbook will dynamically load one of these files based on an `env` label in the incoming alert.
    * **Security Best Practice:** Storing plain-text tokens directly in a Git repository is a significant security risk, and thus we are not doing this. We are relying on the Ansible Automation Platform credential that holds that information, that will be injected into the var file at runtime, and the playbook will use this variables at runtime as well.

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
        oc create token aap-eda-sa -n openshift-monitoring --duration=$((365*24))h
        ```
    The output is a `sha256~...` token. Copy it and save it in a file for a later use in the next moudle.
    
4.  **Create the OpenShift Route manifest + the PrometheusRule for testing:**
    * Create a new file at `extensions/eda/k8s-objects/eda-route.yml` and add the following content. **Replace `<aap-namespace>` with the actual namespace where your AAP instance is installed.**
        ```yaml
        apiVersion: route.openshift.io/v1
        kind: Route
        metadata:
          name: alertmanager-listener
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

    * **Create a Test PrometheusRule for Workshop Demonstration:**
        * Create a new file at `extensions/eda/k8s-objects/test-prometheus-rule.yml` and add the following content. **Replace `<worker-name>` with an actual worker node name from your cluster.**
        ```yaml
        ---
        apiVersion: monitoring.coreos.com/v1
        kind: PrometheusRule
        metadata:
          name: dummy-prometheus-rule
          namespace: openshift-monitoring
        spec:
          groups:
          - name: dummy-prometheus-rule
            rules:
            - alert: NodeHealthCheckFailed
              annotations:
                summary: event driven poc
              expr: |
                max(node_disk_info{instance=~"<worker-name>"}) by(instance)
              for: 10s
              labels:
                severity: custom
        ```

    * **Important Note about the Test PrometheusRule**: This is a **dummy PrometheusRule designed for workshop testing purposes only**. It will trigger alerts almost instantly when applied, allowing you to quickly test your Event-Driven Ansible setup without waiting for real infrastructure issues.
        * The `node_disk_info` metric is always available, so these alerts will fire immediately
        * The `for: 10s` duration means alerts become active after just 10 seconds
        * This allows rapid iteration and testing of your EDA rulebook conditions and remediation workflows
        * **For production scenarios**, refer to the real-world PrometheusRules and monitoring jobs described in the **Appendix** section below

5.  **Create the Rebooter Pod Template:**
    * Create a new file at `extensions/eda/playbooks/templates/reboot_node_pod.yaml.j2` and add the following content:
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

### 2.2 Exercise: Using a Pre-built Decision Environment

For this workshop, we'll use a **pre-built decision environment** that Red Hat provides, which already includes all the necessary components for Event-Driven Ansible.

1.  **Use the Red Hat Supported Decision Environment:**
    * We will use the following pre-built image:
        ```
        registry.redhat.io/ansible-automation-platform-25/de-supported-rhel9:latest
        ```
    * **Why this approach?** This image already includes:
        - The `ansible.eda` collection and all its dependencies
        - All required Python packages for Event-Driven Ansible
        - A certified, supported runtime environment
        - Regular security updates from Red Hat
    * **This is all that's required for our workshop scenario** - no custom building needed!

2.  **When to Build Custom Decision Environments:**
    * The pre-built image works perfectly for most Event-Driven Ansible use cases
    * You may want to build custom decision environments when you need:
        - Additional Ansible collections beyond `ansible.eda`
        - Custom Python packages or dependencies
        - Specific versions of collections for compatibility
        - Custom scripts or tools in your automation environment

3.  **Building Advanced Decision Environments (Optional):**
    * If you want to explore building custom decision environments, you can use `ansible-builder`
    * **Reference Documentation:**
        - [Ansible Builder Documentation](https://ansible-builder.readthedocs.io/)
        - [Red Hat AAP Decision Environment Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/creating_and_using_execution_environments/index)
    * **Basic example** for custom builds:
        ```bash
        # Create requirements.yml with your collections
        # Create execution-environment.yml with your config
        ansible-builder build -f execution-environment.yml -t your-registry/custom-eda-de:latest
        ```

**For this workshop, we'll proceed with the supported Red Hat image** - it provides everything we need for our OpenShift node remediation scenario!

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
                name: reboot-problematic-node
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
          shell: "oc login --insecure-skip-tls-verify=true --token={{ openshift_token }} --server={{ openshift_api }}"

    ##### Just for the workshop
        - name: "Delete Gitea pod only"
          shell: "oc -n gitea delete $(oc get po -n gitea -l app=gitea --no-headers -o name)"

        - name: "Print alert label"
          shell: "echo {{ problematic_node }}"
    #### in our lab we rely on SNO so we cannot really reboot the node......
     
        #- name: "Validation: Don’t reboot node if you have more then 1 NotReady nodes or 3 alert simultaniously"
        #  shell: |
        #    numOfAlertsSimultaniuously=`oc -n openshift-monitoring exec -it $(oc -n openshift-monitoring get pod -l alertmanager=main -o jsonpath='{.items[0].metadata.name}') -- curl -s http://localhost:9093/api/v2/alerts | jq '.[] | select(.labels.alertname=="detect-nfs-stale" or .labels.alertname=="node-health-check") | .fingerprint' | sort -u | wc -l` 
        #    numOfNotReadyNodes=`oc get nodes | grep -E 'NotReady|SchedulingDisabled' | wc -l`
        #    if [ "$numOfAlertsSimultaniuously" -gt 3 ] || [ "$numOfNotReadyNodes" -gt 1 ] ; then
        #      exit 1;
        #    else
        #      exit 0;
        #    fi
        #  register: validations_status
        #  until: validations_status.rc == 0
        #  retries: 10
        #  delay: 600
   
        #- name: "Drain node of its pods"
        #  shell: "oc adm drain {{ problematic_node }} --force --delete-emptydir-data --ignore-daemonsets"
        #  register: drain_status
   
        #- name: "Create rebooter pod manifest from template"
        #  template:
        #    src: ../../templates/reboot_node_pod.yaml.j2
        #    dest: "/tmp/reboot_node_pod-{{ problematic_node }}.yaml"
   
        #- name: "Apply the rebooter pod manifest to trigger the node reboot"
        #  shell: "oc apply -f /tmp/reboot_node_pod-{{ problematic_node }}.yaml"
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

## Appendix: Production-Ready Monitoring Components

This workshop uses simplified test alerts for demonstration purposes. For real-world implementations of node health monitoring and NFS stale detection, refer to these production-ready components:

### Real PrometheusRules

Comprehensive PrometheusRules for production node health monitoring:
- **Source**: [EDA-Real-PrometheusRules.yaml](https://gist.githubusercontent.com/tommeramber/b9208bfe558f8119bd897c63a599ef9c/raw/4756e02c6b6f575ffac20ff9bdc2b5a61b58c6e9/EDA-Real-PrometheusRules.yaml)
- Includes rules for:
  - PVC over-utilization detection
  - Node health check job completion monitoring
  - NFS stale mount detection

### Monitoring CronJobs

Automated health check jobs that generate the alerts monitored by the PrometheusRules:
- **Source**: [EDA-CronJobs.yaml](https://gist.githubusercontent.com/tommeramber/08ad6ac3af85face1982aa9a3526bbc9/raw/727c726e442089fed0c1c4e2c235e16f12a93b92/EDA-CronJobs.yaml)
- Includes:
  - `detect-nfs-stale`: Periodic NFS mount health checks
  - `node-health-check`: Comprehensive node health validation

### Additional Resources

For a detailed technical deep-dive into the concepts and implementation patterns used in this workshop:
- **Article**: ["Optimizing Cloud Native Operations Series - Part 3: About the Rulebook"](https://medium.com/@tamber/optimizing-cloud-native-operations-series-part-3-about-the-rulebook-d57d828254da)

These resources provide the foundation for implementing robust, production-ready Event-Driven Ansible solutions in enterprise OpenShift environments.
