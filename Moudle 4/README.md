    ## Module 4: Triggering and Observing the Automation

### 4.1 Exercise: Configure Global Alert Labeling

1.  **Create a file named `extensions/eda/k8s-objects/cluster-monitoring-config.yml`** with the following content:
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: cluster-monitoring-config
      namespace: openshift-monitoring
    data:
      config.yaml: |
        prometheusK8s:
          externalLabels:
            env: dev # Or 'prd', depending on the cluster
    ```
2.  **Apply the ConfigMap to your cluster:**
    ```bash
    oc apply -f extentions/eda/k8s-objects/cluster-monitoring-config.yml
    ```

### 4.2 Exercise: Configuring OpenShift Alertmanager

1.  From the OCP UI -> `Administration` -> `Cluster Settings` -> `Configuration` -> `AlertManager`
2.  **Add a new receiver and route** pointing to the webhook URL from step 3.3:
     ```yaml
     ...
     receivers:
     - name: EDA
       webhook_configs:
       - url: "https://<echo $ROUTE>/alerts"
         send_resolved: false
     ...
     route:
       routes:
         - reciever: EDA
           match_re:
             severity: custom
           continue: true
     ```

### 4.3 Exercise: Generating a Test Alert

1.  **Edit the file named `extensions/eda/k8s-objects/test-prometheus-rule.yml`** with the worker node from the cluster you want the automation to reboot.
2.  **Apply the rule:**
    ```bash
    oc apply -f extensions/eda/k8s-objects/test-prometheus-rule.yml
    ```
3.  **Save your test alert to Git (Optional but good practice):**
    ```bash
    # Move the file into your repo structure
    git add .
    git commit -m "Add test alert for node health"
    git push
    ```

### 4.4 Exercise: Observing the Event and Action

1.  **In the EDA UI:** Watch the `History` tab of your `alertmanager-listener` activation.
2.  **In the AAP UI:** Watch the `Jobs` page for the "Drain and Reboot Unhealthy Node" job to launch.

