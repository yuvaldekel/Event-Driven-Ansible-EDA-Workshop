
## Module 0: The Use Case

The scenario for our automation involves an OpenShift cluster that uses Trident as its storage provider, connected to a NetApp backend. This connection passes through a problematic and unstable firewall which, due to a lack of memory and an excess of rules, experiences frequent "hiccups."

These hiccups cause short, intermittent disconnections between some OpenShift nodes and the NetApp storage. This behavior results in unresponsive PVCs (broken volume mounts) and pending PVs for new requests. Critically, the node itself remains scheduleable and the Kubelet reports as healthy, so the Kubernetes control plane is unaware that the node has a serious storage issue that requires a reboot to mitigate.

This issue occurs roughly once a day. The customer's SRE team knows how to detect it: they run a CronJob in the `openshift-monitoring` namespace every five minutes. This job attempts to run a pod with a PVC, and if it succeeds, the pod is deleted. A corresponding `PrometheusRule` checks if this job fails. A failure indicates that the storage issue has occurred on a node.

The goal of this workshop is to use Event-Driven Ansible to fully automate the solution, replacing the need for an SRE team member to manually detect the alert and reboot the affected node.

