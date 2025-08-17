# 1-Day Workshop: Mastering Event-Driven Ansible with OpenShift

**Instructor:** Tommer Amber, Red Hat Managing Architect

**Audience:** OpenShift and Ansible administrators familiar with Ansible Automation Platform (AAP) but new to Event-Driven Ansible (EDA).

**Workshop Goal:** By the end of this workshop, you will be able to design, configure, and implement Event-Driven Ansible to automate responses to events from multiple OpenShift clusters, enhancing your operational efficiency.

**Software Prerequisites:**

* An OpenShift cluster with Ansible Automation Platform (AAP) 2.5 or higher installed and running.
* `oc` and `ansible-builder` command-line tools installed on your local machine.
* A personal GitHub account.
* A user account on Quay.io (or another container registry) to store the Decision Environment image.

## Initial Lab Setup: Forking the Workshop Repository

Before we begin, you need to get a personal copy of the workshop's source code. This involves forking the main repository and then cloning your fork to your local machine. This ensures you have your own version of the code to modify and use with your AAP instance.

1.  **Fork the Workshop Repository:**
    * Navigate to the main workshop repository URL: `https://github.com/tommeramber/eda-openshift-workshop`
    * Click the **"Fork"** button in the top-right corner of the page.
    * When prompted, select your personal GitHub account as the destination for the fork. This will create a copy of the repository under your account (`https://github.com/YOUR_USERNAME/eda-openshift-workshop`).

2.  **Clone Your Forked Repository:**
    * On your personal fork's GitHub page, click the green **"< > Code"** button.
    * Ensure the "HTTPS" tab is selected, and copy the URL.
    * Open a terminal on your local machine and run the following command, replacing the URL with the one you just copied:
        ```bash
        git clone [https://github.com/YOUR_USERNAME/eda-openshift-workshop.git](https://github.com/YOUR_USERNAME/eda-openshift-workshop.git)
        ```

3.  **Navigate into the Repository Directory:**
    * Once the clone is complete, move into the new directory:
        ```bash
        cd eda-openshift-workshop
        ```

You now have a local copy of all the necessary playbooks, rulebooks, and templates, and it's linked to your personal GitHub repository. You are ready to begin the workshop exercises.
