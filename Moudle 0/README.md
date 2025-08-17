
## Module 0: The Use Case

The scenario for our automation involves an OpenShift cluster that uses Trident as its storage provider, connected to a NetApp backend. This connection passes through a problematic and unstable firewall which, due to a lack of memory and an excess of rules, experiences frequent "hiccups."

These hiccups cause short, intermittent disconnections between some OpenShift nodes and the NetApp storage. This behavior results in unresponsive PVCs (broken volume mounts) and pending PVs for new requests. Critically, the node itself remains scheduleable and the Kubelet reports as healthy, so the Kubernetes control plane is unaware that the node has a serious storage issue that requires a reboot to mitigate.

This issue occurs roughly once a day. The customer's SRE team knows how to detect it: they run a CronJob in the `openshift-monitoring` namespace every five minutes. This job attempts to run a pod with a PVC, and if it succeeds, the pod is deleted. A corresponding `PrometheusRule` checks if this job fails. A failure indicates that the storage issue has occurred on a node.

The goal of this workshop is to use Event-Driven Ansible to fully automate the solution, replacing the need for an SRE team member to manually detect the alert and reboot the affected node.

<img width="682" height="262" alt="Screenshot From 2025-08-17 11-48-40" src="https://github.com/user-attachments/assets/6dfcb955-52ab-45a7-920b-44c7c84fecf6" />

<img width="896" height="377" alt="Screenshot From 2025-08-17 12-01-33" src="https://github.com/user-attachments/assets/71d45d8b-e73e-4c82-bc43-e6f779de4ff0" />

<img width="631" height="579" alt="Screenshot From 2025-08-17 12-03-32" src="https://github.com/user-attachments/assets/a4b8464b-e58c-497d-9544-612a03090c4a" />
