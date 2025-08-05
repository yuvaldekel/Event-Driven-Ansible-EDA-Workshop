# 1-Day Hands-On Workshop: Event-Driven Ansible for OpenShift

**Instructor:** Tommer Amber, Red Hat Managing Architect

**Duration:** 1 Day

**Audience:** OpenShift Administrators, Automation Engineers, SREs.

**Prerequisites:**
* Familiarity with Ansible Playbooks, Inventories, and Job Templates.
* Basic understanding of OpenShift and `oc` CLI concepts.
* A running OpenShift Cluster with Ansible Automation Platform (AAP) 2.5+ operator installed and running.
* `oc` and `ansible-rulebook` CLI tools installed on the trainee's workstation.
* Access to a Git repository (like GitHub or GitLab) for the CaaC lab.

---

## Workshop Agenda

| Time          | Module                                                    | Lab                                               |
|---------------|-----------------------------------------------------------|---------------------------------------------------|
| 9:00 - 9:30   | **Module 1: Introduction to Event-Driven Automation** | -                                                 |
| 9:30 - 10:30  | **Module 2: EDA Core Components & Architecture** | -                                                 |
| 10:30 - 10:45 | *Break* | -                                                 |
| 10:45 - 12:00 | **Module 3: Deep Dive into Rulebooks** | **Lab 1: Your First Rulebook (Webhook)** |
| 12:00 - 1:00  | *Lunch* | -                                                 |
| 1:00 - 2:30   | **Module 4: Integrating EDA with OpenShift** | **Lab 2: Responding to OpenShift Pod Failures** |
| 2:30 - 3:30   | **Module 5: Configuration-as-Code for EDA** | **Lab 3: Managing Rulebooks with Git** |
| 3:30 - 3:45   | *Break* | -                                                 |
| 3:45 - 4:30   | **Module 6: Advanced Scenarios & Best Practices** | -                                                 |
| 4:30 - 5:00   | **Wrap-up & Open Q&A** | -                                                 |

---

## Module 1: Introduction to Event-Driven Automation

**(Information Section)**

Welcome to the Event-Driven Ansible (EDA) workshop!

For years, we've used Ansible to *make* changes happen. We run a playbook, and the state of our infrastructure changes. This is a **desired state** model, and it's incredibly powerful. But what about responding to changes that happen *outside* of our control?

* A pod in OpenShift enters a `CrashLoopBackOff` state.
* A web server's performance metrics cross a critical threshold.
* A security scanner detects a new vulnerability.
* A developer pushes new code to a Git repository.

These are **events**. Traditionally, addressing them required manual intervention or complex, custom-coded monitoring scripts.

**Event-Driven Ansible** flips the model. Instead of you telling Ansible when to run, Ansible *listens* for events from your IT landscape and *reacts* automatically based on rules you define. It connects event intelligence to your existing, trusted automation.

With EDA, you can build fully automated, closed-loop remediation systems. You can create workflows that not only detect an issue on a remote OpenShift cluster but also log in as a privileged user and execute a playbook to fix it, all without a human touching a keyboard.

In this workshop, we will explore how the EDA capabilities within Ansible Automation Platform 2.5 can be used to achieve this, focusing specifically on monitoring and managing remote OpenShift clusters.

---

## Module 2: EDA Core Components & Architecture

**(Information Section)**

To understand EDA, we need to know its three main components:

1.  **Event Sources:** These are plugins that connect to external systems and listen for events. EDA comes with a rich set of source plugins for systems like Kafka, Prometheus, generic webhooks, and, most importantly for us, Kubernetes/OpenShift. We will primarily use the `redhat.eda.k8s` source plugin.

2.  **Rulebooks:** This is the heart of EDA. A Rulebook is a YAML file, much like a playbook, that defines the logic of your event-driven automation. It links a specific event from a source to a specific action. The structure is simple: **IF** `this event` happens, **THEN** `do this action`.

3.  **Actions:** An action is what EDA executes when the conditions of a rule are met. Common actions include:
    * `run_playbook`: The most powerful action, which triggers an Ansible Playbook.
    * `run_job_template`: The AAP equivalent, which launches a pre-configured Job Template.
    * `set_fact`: To create a new fact from event data for use in subsequent rules.
    * `post_event`: To send a new, modified event back to the rulebook engine.

**Architecture within AAP on OpenShift:**

When you deploy AAP 2.5+, you get a new component: the **EDA Controller**. This is where you configure and manage your event-driven automation.

* You create a **Project** that points to your rulebooks (typically in Git).
* You define an **EDA Decision Environment**, which is a container image containing `ansible-rulebook` and all necessary collections (like `redhat.eda`).
* You create an **EDA Rulebook Activation**, which links a rulebook from your project with an event source and runs it as a persistent pod within your OpenShift cluster. This pod is what actively listens for events.

When an event occurs, the Rulebook Activation pod processes it, and if a rule matches, it uses the AAP API to launch a Job Template to perform the remediation.

---

## Module 3: Deep Dive into Rulebooks

**(Information Section)**

Let's break down the structure of a Rulebook. It's a YAML file with three main sections: `sources`, `rules`, and `actions`.

```yaml
---
- name: A simple rulebook to listen for webhooks
  hosts: all

  sources:
    - ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000

  rules:
    - name: Say hello when a message arrives
      condition: event.payload.message == "hello"
      action:
        debug:
          msg: "Hello, world! The webhook payload was: {{ event.payload }}"

    - name: Run a playbook for a specific task
      condition: event.payload.task == "deploy"
      action:
        run_playbook:
          name: playbooks/deploy_app.yml
```

* **`name`**: A descriptive name for the rulebook.
* **`hosts`**: Defines the target hosts for the actions. `all` is common for EDA.
* **`sources`**: A list of event source plugins to use. Here, we're using the built-in webhook listener.
* **`rules`**: A list of rules. The engine evaluates rules in order for each event.
    * **`name`**: A descriptive name for the rule.
    * **`condition`**: A Jinja2 expression. The rule's action only runs if this expression evaluates to `true`. The event data is available in the `event` variable (or `events` for multiple events).
    * **`action`**: What to do when the condition is met. Here we see `debug` and `run_playbook`.

### **Lab 1: Your First Rulebook (Webhook)**

**Objective:** Create and run a simple rulebook locally that listens for a generic webhook and prints a message. This teaches the fundamental structure of a rulebook without the complexity of AAP.

**Steps:**

1.  **Create a Project Directory:**
    ```bash
    mkdir eda-lab-1
    cd eda-lab-1
    ```

2.  **Create the Rulebook:**
    Create a file named `rulebooks/webhook-example.yml`:
    ```yaml
    ---
    - name: Listen for a simple webhook
      hosts: localhost

      sources:
        - ansible.eda.webhook:
            host: 0.0.0.0
            port: 5001 # Use a unique port

      rules:
        - name: Process a new user registration
          condition: event.payload.action == "register" and event.payload.user is defined
          action:
            debug:
              msg: "A new user named '{{ event.payload.user }}' has registered. Welcome!"

        - name: Catch all other events
          condition: true
          action:
            debug:
              msg: "Received an unhandled event: {{ event.payload }}"
    ```

3.  **Run the Rulebook:**
    Use the `ansible-rulebook` CLI to start the listener. You'll need the `ansible.eda` collection installed (`ansible-galaxy collection install ansible.eda`).
    ```bash
    ansible-rulebook --rulebook rulebooks/webhook-example.yml -i inventory.yml
    ```
    *(Note: An empty `inventory.yml` file is fine for this lab)*

4.  **Trigger the Event:**
    Open a second terminal and use `curl` to send a JSON payload to the webhook listener.

    * **Trigger the first rule:**
        ```bash
        curl -H "Content-Type: application/json" -X POST \
        -d '{"action": "register", "user": "alice", "timestamp": "2024-08-05T12:00:00Z"}' \
        http://localhost:5001/endpoint
        ```
        You should see the "new user" message in your `ansible-rulebook` terminal.

    * **Trigger the second rule:**
        ```bash
        curl -H "Content-Type: application/json" -X POST \
        -d '{"action": "login", "user": "bob"}' \
        http://localhost:5001/endpoint
        ```
        You should see the "unhandled event" message.

**Congratulations!** You've successfully created and tested your first Event-Driven Ansible rulebook.

---

## Module 4: Integrating EDA with OpenShift

**(Information Section)**

Now for the main event: listening to OpenShift. The `redhat.eda.k8s` source plugin is our tool for this. It connects to the Kubernetes API server and watches for changes to resources like Pods, Deployments, Nodes, etc.

A common use case is to watch for Pods that are not in a healthy state. When a Pod enters a state like `Failed` or `CrashLoopBackOff`, the Kubernetes API generates an event. Our rulebook can listen for this event.

The event payload from the `k8s` source is rich with information, including:
* `event.type`: e.g., `ADDED`, `MODIFIED`, `DELETED`.
* `event.meta.namespace`: The OpenShift project where the event occurred.
* `event.meta.name`: The name of the resource (e.g., the Pod name).
* `event.reason`: A short string indicating why the event was generated (e.g., `Failed`, `BackOff`).
* `event.note`: A longer description of the event.

We can use these fields in our rule conditions to trigger highly specific remediation playbooks.

### **Lab 2: Responding to OpenShift Pod Failures**

**Objective:** Configure AAP to automatically launch a remediation Job Template when a Pod in a specific namespace fails. This lab demonstrates the full, end-to-end workflow on the platform.

**Prerequisites:**
* You are logged into your AAP Web UI as an administrator.
* You have a project/namespace on your OpenShift cluster where you can create pods (e.g., `eda-test-project`).

**Steps:**

1.  **Create the Remediation Playbook:**
    In a Git repository that your AAP instance can access, create `playbooks/fix-broken-pod.yml`:
    ```yaml
    ---
    - name: Remediate a failed pod
      hosts: localhost
      connection: local
      gather_facts: false

      vars:
        # These vars will be passed from the EDA event
        pod_name: "{{ pod_name | default('unknown_pod') }}"
        namespace: "{{ namespace | default('unknown_namespace') }}"

      tasks:
        - name: Log the failure event
          lineinfile:
            path: "/tmp/eda_pod_failures.log"
            line: "FAILURE DETECTED: Pod '{{ pod_name }}' in namespace '{{ namespace }}' failed at {{ ansible_date_time.iso8601 }}."
            create: true

        - name: Gather details about the failed pod
          kubernetes.core.k8s_info:
            api_version: v1
            kind: Pod
            name: "{{ pod_name }}"
            namespace: "{{ namespace }}"
          register: pod_info

        - name: Print the pod logs for analysis
          debug:
            msg: "Logs from failed pod {{ pod_name }}:\n{{ lookup('kubernetes.core.k8s_log', pod_name=pod_name, namespace=namespace) }}"
          when: pod_info.resources | length > 0

        - name: Delete the failed pod to allow ReplicaSet to recreate it
          kubernetes.core.k8s:
            state: absent
            api_version: v1
            kind: Pod
            name: "{{ pod_name }}"
            namespace: "{{ namespace }}"
          when: pod_info.resources | length > 0
    ```

2.  **Create the Job Template in AAP:**
    * **Name:** `EDA - Remediate Failed Pod`
    * **Inventory:** `Demo Inventory` (or any suitable inventory)
    * **Project:** The project pointing to your Git repo.
    * **Playbook:** `playbooks/fix-broken-pod.yml`
    * **Credentials:** A credential that can interact with the *remote* OpenShift cluster (e.g., a "Kubernetes API Token" credential type with a service account token).
    * **Enable "Prompt on Launch"** for `Extra Variables`. This is crucial.

3.  **Create the EDA Rulebook:**
    In the same Git repository, create `rulebooks/monitor-openshift-pods.yml`:
    ```yaml
    ---
    - name: Monitor and remediate failed pods in OpenShift
      hosts: localhost

      sources:
        - redhat.eda.k8s:
            api_version: v1
            kind: Event
            namespace: eda-test-project # IMPORTANT: Watch a specific namespace

      rules:
        - name: A pod has failed and needs remediation
          # We look for events with reason 'Failed' or 'BackOff'
          condition: >
            event.reason in ["Failed", "BackOff"] and
            event.type == "MODIFIED" and
            event.object.kind == "Pod"
          action:
            run_job_template:
              name: "EDA - Remediate Failed Pod"
              organization: "Default"
              job_args:
                extra_vars:
                  # Pass data from the event to the Job Template
                  pod_name: "{{ event.object.metadata.name }}"
                  namespace: "{{ event.object.metadata.namespace }}"
    ```

4.  **Create the Rulebook Activation in AAP:**
    * Go to `Event-Driven Ansible` -> `Rulebook Activations` -> `Create Rulebook Activation`.
    * **Name:** `OpenShift Pod Monitor`
    * **Project:** Your Git project.
    * **Rulebook:** `rulebooks/monitor-openshift-pods.yml`
    * **Decision Environment:** `Event-Driven Ansible 2.5` (or your custom DE).
    * **Restart Policy:** `Always`.
    * Enable the activation after creating it. It will start a pod to listen for events.

5.  **Trigger the Failure:**
    * Log in to your OpenShift cluster via the `oc` CLI.
    * Create the target namespace: `oc new-project eda-test-project`
    * Create a pod that is designed to fail. Create `fail-pod.yml`:
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: bad-pod
        spec:
          containers:
          - name: busybox
            image: busybox:1.28
            command: ["/bin/sh", "-c", "exit 1"] # This command exits immediately with an error
          restartPolicy: Always # This will cause CrashLoopBackOff
        ```
    * Apply it: `oc apply -f fail-pod.yml -n eda-test-project`

6.  **Observe the Automation:**
    * Watch the pod status: `oc get pods -n eda-test-project -w`
    * You will see it go from `Running` -> `Error` -> `CrashLoopBackOff`.
    * In the AAP UI, go to `Jobs`. You should see the `EDA - Remediate Failed Pod` job launch automatically.
    * Check the job output. It will log the failure and then delete the pod.
    * Back in your `oc` terminal, you'll see the `bad-pod` is terminated and then recreated by OpenShift.

---

## Module 5: Configuration-as-Code for EDA

**(Information Section)**

Managing automation assets through a UI is great for getting started, but for enterprise-scale operations, we need a more robust, auditable, and collaborative approach. This is where Configuration-as-Code (CaaC) comes in.

The idea is simple: your **Git repository is the single source of truth**. All of your EDA configurations—Rulebooks, Playbooks, and even the AAP configuration itself (using `awx-cli`)—are defined as code.

**Benefits:**
* **Versioning:** Every change is a Git commit. You can see who changed what, when, and why.
* **Auditing:** Your entire automation configuration is auditable.
* **Collaboration:** Teams can collaborate on automation using familiar pull/merge request workflows.
* **Consistency:** You can easily replicate your EDA setup across multiple AAP instances (e.g., dev, stage, prod).

AAP is designed for this. Its Projects automatically sync with Git repositories, meaning any change you merge to your `main` branch is immediately reflected and ready to be used in the platform.

### **Lab 3: Managing Rulebooks with Git**

**Objective:** Modify an existing rulebook in Git and see the change automatically picked up and used by the Rulebook Activation in AAP.

**Steps:**

1.  **Update the Rulebook in Your Git Repo:**
    * Clone the Git repository you used in Lab 2 to your local machine.
    * Edit `rulebooks/monitor-openshift-pods.yml`. Let's add a new rule to also catch pods that are stuck in the `Pending` state for too long (this is a conceptual rule, as "too long" requires more complex logic, but we'll simulate it).
    ```yaml
    ---
    - name: Monitor and remediate pod issues in OpenShift
      hosts: localhost

      sources:
        - redhat.eda.k8s:
            api_version: v1
            kind: Event
            namespace: eda-test-project

      rules:
        - name: A pod has failed and needs remediation
          condition: >
            event.reason in ["Failed", "BackOff"] and
            event.type == "MODIFIED" and
            event.object.kind == "Pod"
          action:
            run_job_template:
              name: "EDA - Remediate Failed Pod"
              organization: "Default"
              job_args:
                extra_vars:
                  pod_name: "{{ event.object.metadata.name }}"
                  namespace: "{{ event.object.metadata.namespace }}"

        # NEW RULE ADDED
        - name: A pod seems stuck in Pending state
          condition: event.reason == "Scheduled" and event.type == "NORMAL" and event.object.kind == "Pod"
          action:
            debug:
              msg: "FYI: Pod {{ event.object.metadata.name }} was just scheduled. Monitoring for potential 'Pending' state issues."
    ```

2.  **Commit and Push the Change:**
    ```bash
    git add rulebooks/monitor-openshift-pods.yml
    git commit -m "feat: Add rule to watch for Pending pods"
    git push
    ```

3.  **Sync the Project in AAP:**
    * In the AAP UI, navigate to `Projects`.
    * Find your project and click the "Sync" button. AAP will pull the latest commit from your repository.

4.  **Restart the Rulebook Activation:**
    * For the changes to take effect, the Rulebook Activation pod needs to be restarted to use the new version of the rulebook.
    * Go to `Event-Driven Ansible` -> `Rulebook Activations`.
    * Find your `OpenShift Pod Monitor` activation, and use the toggle to disable and then re-enable it. This forces AAP to pull the latest synced rulebook and restart the listener.

5.  **Test the New Rule:**
    * Create a new pod in the `eda-test-project` namespace.
    * Check the logs of the `OpenShift Pod Monitor` activation pod in the AAP UI (or via `oc logs` in the `aap` namespace).
    * You should see the new "FYI: Pod ... was just scheduled" debug message, proving that your CaaC workflow was successful.

---

## Module 6: Advanced Scenarios & Best Practices

**(Information Section)**

What we've done today is the foundation. Now, let's consider how to expand on it.

* **Multi-Cluster Remediation:** The Job Template we built can be made more generic. By parameterizing the cluster API endpoint and credentials, a single Job Template could remediate issues on any of your remote OpenShift clusters. The event source itself could come from a centralized logging or monitoring platform (like Splunk or Dynatrace) that aggregates events from all clusters.

* **Intelligent Remediation:** Instead of just deleting a pod, your playbook could be much smarter.
    * It could scale up a deployment if CPU/memory pressure is high.
    * It could analyze logs for specific error messages and apply a targeted fix.
    * It could open a ticket in ServiceNow or post a message to a Slack channel *before* taking action, allowing for human oversight.

* **Event Throttling:** What if 100 pods fail at once? You don't want to launch 100 remediation jobs. EDA has a `throttle` setting in rules to prevent this "event storm." You can configure it to run a rule once, and then ignore subsequent matching events for a defined period.

* **Security:** Always follow the principle of least privilege. The service account token used by the Job Template should have the *minimum* permissions required to fix the issue on the remote cluster (e.g., only permissions to `get` and `delete` pods in a specific namespace).

---

## Wrap-up & Open Q&A

Today we've seen how Event-Driven Ansible transforms Ansible from a tool for executing commands to a platform for building intelligent, automated response systems.

**Key Takeaways:**
1.  EDA listens for events and reacts based on rules you define in a **Rulebook**.
2.  The **EDA Controller** in AAP on OpenShift is the engine that runs your event-driven automation.
3.  You can easily integrate with **OpenShift** to listen for resource changes and trigger remediation.
4.  Managing your rulebooks and playbooks in **Git (CaaC)** is the best practice for a scalable, auditable, and collaborative automation strategy.

This is just the beginning. Think about the repetitive manual tasks your teams perform today in response to system alerts. Which of those can you automate away with EDA?

**Thank you!**

