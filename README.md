# OpenShift TCPDUMP Collector DaemonSet
This repository provides an OpenShift DaemonSet to facilitate network traffic capture (tcpdump) across all worker nodes in your cluster. This solution is designed for network diagnostics, offering continuous capture, daily file rotation, and automated cleanup, while addressing common operational challenges in containerized environments.

# Features
- Node-Wide Deployment: Deploys one tcpdump collector pod to each eligible worker node in your OpenShift cluster.
- Interface-Specific Capture: Configured to capture all network traffic on the enp3s0 network interface. This can be easily modified to target other interfaces or any (for all interfaces).
- Host Network and PID Access: The pod runs with hostNetwork: true and hostPID: true to enable comprehensive network traffic capture and access to host process information.
- Privileged Security Context: Requires privileged security context and NET_ADMIN, NET_RAW capabilities to perform network packet capture.
- Persistent Storage on Host: Captured .pcap files are stored directly on the host node's filesystem at /var/log/tcpdump using a hostPath volume.
- SELinux Context Management: An initContainer is included to ensure the /var/log/tcpdump directory on the host has the correct SELinux context (svirt_sandbox_file_t), allowing the container to write to it.
- Daily Rotation and Compression: The tcpdump process runs continuously for 24 hours. At midnight, the current capture is stopped, compressed using gzip (e.g., tcpdump-YYYYMMDD-HHMMSS.pcap.gz), and a new capture begins.
- Automated Cleanup: Old compressed .pcap.gz files older than 7 days are automatically deleted to manage disk space. This retention period is configurable within the YAML.

# Prerequisites
For the tcpdump-collector DaemonSet to function correctly, a Service Account with the privileged Security Context Constraint (SCC) must be configured in your OpenShift project. This grants the necessary permissions for the pod to perform host-level network operations.

Execute the following commands in your OpenShift cluster, replacing <your-namespace> with your target project:
~~~
oc project <your-namespace>
oc create sa tcpdump-collector-sa
oc adm policy add-scc-to-user privileged -z tcpdump-collector-sa
~~~

# Deployment
Clone this repository:
~~~
git clone https://github.com/osamazaid/openshift-tcpdump-collector.git
cd openshift-tcpdump-collector
~~~
Ensure you are logged into your OpenShift cluster and are in the correct project:
~~~
oc login ...
oc project <your-namespace>
~~~
Apply the DaemonSet:
~~~
oc apply -f openshift/tcpdump-daemonset.yaml
~~~

# Monitoring and Verification
After deployment, it is crucial to monitor the DaemonSet's status and verify its operation:

Check Pod Status: Confirm that tcpdump-collector pods are running on your worker nodes:
~~~
oc get pods -l app=tcpdump-collector -n <your-namespace>
~~~
Review Pod Logs: Examine the logs of a running pod to ensure tcpdump is capturing traffic and the rotation script is executing:
~~~
oc logs <tcpdump-collector-pod-name> -c tcpdump-container -n <your-namespace>
~~~
Verify Files on Host: SSH into one of your worker nodes and inspect the /var/log/tcpdump directory to confirm that .pcap and .pcap.gz files are being created and rotated:
~~~
ls -lh /var/log/tcpdump/
~~~
# Estimated Disk Usage
Based on observed network traffic patterns (approximately 27 MB/minute of uncompressed data on the enp3s0 interface), an uncompressed tcpdump file on a single worker node can grow to approximately 38 GB over a 24-hour period.

However, the DaemonSet includes gzip compression, which is highly effective for tcpdump files, significantly reducing the actual disk space consumed (typically to 10-15 GB or less). It is imperative to ensure that the /var/log partition on your worker nodes has sufficient free space to accommodate these files, considering the 7-day retention policy.

# Customization
You can customize the DaemonSet by editing the openshift/tcpdump-daemonset.yaml file:

INTERFACE: Change "enp3s0" to another specific interface (e.g., "eth0", "bond0") or "any" to capture on all interfaces.

FILE_PREFIX: Modify the prefix for the capture files.

Retention Period: Adjust -mtime +7 in the find commands to keep files for more or fewer days.

# License
This project is licensed under the MIT License - see the LICENSE file for details.
