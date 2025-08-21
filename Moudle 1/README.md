## Module 1: Introduction to Event-Driven Ansible (EDA)

### 1.1 Information: What is Event-Driven Ansible?

Event-Driven Ansible (EDA) is a component of the Ansible Automation Platform that provides an event-driven automation solution. It's designed to process events from various sources and trigger automated actions in response.

* **EDA vs. Ansible Automation Platform (AAP):** While AAP is primarily a platform for executing imperative, user-initiated automation (running playbooks on a schedule or on-demand), EDA introduces a reactive model. It listens for events from your IT environment and automatically triggers the appropriate response. Think of AAP as the "doer"; it executes the automation tasks. EDA is the "listener" that constantly watches for specific signals (events) and then tells the "doer" (AAP) which playbook to run. This combination allows for a much more dynamic and responsive automation strategy, where actions are taken the moment a problem is detected, not just when a human or a schedule decides.

* **Decision Environments vs. Execution Environments:**
    * **Execution Environments (EEs):** You are likely familiar with EEs in AAP. They are container images that provide a standardized and portable environment for running Ansible Playbooks. They package Ansible, Python, collections, and all necessary dependencies.
    * **Decision Environments (DEs):** DEs are the equivalent for EDA. They are container images that contain the `ansible-rulebook` CLI, the necessary event source plugins (like `ansible.eda.alertmanager`), and any other dependencies required to interpret and act upon incoming events. We will be creating a lean DE to ensure our EDA is efficient.

* **Rulebooks vs. Playbooks:**
    * **Playbooks:** A list of tasks to be executed in a specific order on a defined set of hosts. They are the core of Ansible's configuration management and orchestration capabilities.
    * **Rulebooks:** A set of rules that define what actions to take when specific events occur. A rulebook connects an event source to an action. The action is often to run a playbook in AAP. A rulebook has a clear structure:
        1.  **Sources:** Defines where the events are coming from. This is handled by a source plugin, like `ansible.eda.alertmanager` for Prometheus or `ansible.eda.kafka` for Kafka topics.
        2.  **Rules:** A set of conditions that are evaluated against the incoming event data. Each rule has a `condition` and an `action`.
        3.  **Condition:** A logical expression that checks the content of the event. For example, `event.alert.labels.severity == "critical"`. If the condition is true, the action is triggered.
        4.  **Action:** When a condition is met, the specified actions are executed. A primary action is `run_job_template`, which triggers a predefined Ansible Job Template in AAP. Crucially, the rulebook passes the entire event data (`event.alert`) as `extra_vars` to the Ansible playbook, typically named `payload`, allowing the playbook to dynamically use the alert's context. You must also specify the `organization` the job template belongs to.

* **Event Sources:** These are plugins that connect EDA to external event producers. EDA can listen to a wide variety of sources, including Kafka, webhooks, and, as we'll see today, Prometheus Alertmanager. The `ansible.eda.alertmanager` source plugin allows our rulebook to receive alerts directly from OpenShift's built-in monitoring.

### 1.2 Exercise: Create Project & Install Operator

1.  **Log in to the OpenShift Cluster and create an aap Namespace** with a cluster-admin user.
```
OCP_URL="<GET FROM INSTRATCTOR>"
ADMIN_PASSWORD="<GET FROM INSTRATCTOR>"
oc login -u admin -p ${ADMIN_PASSWORD} ${OCP_URL}
oc create ns aap
```
2. **Install the AAP Operator (2.5 cluster scoped)** from the Openshift UI

    1. `OperatorHub` -> `Ansible Automation Platform`
    2. `Channel` == `stable-2.5-cluster-scoped`
    3. Press `Install`
    4. `Installed namespace` = `Operator recommended Namespace: aap`
    5. Create new `AnsibleAutomationPlatform` object (use default settings)
    6. Navigate to `Workloads` => `Pods` and wait for all pods to run successfully (10min +/-)
    7. Once they are all up - make sure you see 3 routes:
        * **main aap instance route**
        * **controller route**
        * **eda route**
        * **hub route**
    8. Press the main aap instance route
        * Username: admin
        * Password: from `<aap-instance>-admin-password`
      
<img width="806" height="909" alt="image" src="https://github.com/user-attachments/assets/9e852a34-f638-4a2e-a7b2-59a96340ee25" />

