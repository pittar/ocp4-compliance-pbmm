# OpenShift Compliance Operator - Mapping to ITSG-33

THe OpenShift [Compliance Operator](https://docs.openshift.com/container-platform/4.9/security/compliance_operator/compliance-operator-supported-profiles.html) offers a number of scanning profiles.  The ones that are most relevant for Canadian Government departments that need to prove compliance of their platforms are:

* **ocp4-moderate** - NIST 800-53 Moderate-Impact Baseline for Red Hat OpenShift - Platform level
* **ocp4-moderate-node** - NIST 800-53 Moderate-Impact Baseline for Red Hat OpenShift - Node level
* **rhcos4-moderate** - NIST 800-53 Moderate-Impact Baseline for Red Hat Enterprise Linux CoreOS

You can see the [NIST-800-53 controls here](https://csrc.nist.gov/Projects/risk-management/sp800-53-controls/release-search#!/800-53).

The reason these scans are relevant is that the [ITSG-33](https://cyber.gc.ca/en/guidance/it-security-risk-management-lifecycle-approach-itsg-33) standard maps almost 1:1 to NIST-800-53.  For this reason, running the above scans on your OpenShift environment can greatly improve your ability to remediate issues and provide an evidence package to support your authority to operate.

## Installing the Compliance Operator

The Compliance Operator can be installed in a few different ways:

1. Through OperatorHub: An OpenShift cluster administrator can login to OpenShift, search the integrated "OperatorHub" for "Compliance Operator", then accept all defaults to install the Compliance Operator into the recommended namespace.
2. [Red Hat Advanced Cluster Management for Kubernetes](https://www.redhat.com/en/technologies/management/advanced-cluster-management) can be used to create a "Policy" that automatically installs the Compliance Operator on all managed OpenShift clusters.
3. GitOps: A GitOps approach can be taken in order to use a tool such as [Argo CD (OpenShift GitOps)](https://argo-cd.readthedocs.io/en/stable/) to install the Compliance Operator.

Going through each option in detail is out of scope for this document.  For now, you can either take the 1st approach and install the Compliance Operator through the OpenShift Admin UI, or, if you are already logged in to your cluster as a `cluster-admin`, you can execute the following command to install the Compliance Operator with default settings.

```
oc apply -k https://github.com/redhat-cop/gitops-catalog/compliance-operator/operator/overlays/release-0.1
```

After a few minutes, the Compliance Operator will be running in a new namespace called `openshift-compliance`.

You will know when the operator is ready as you will now be able to view the different profiles that are available:

```
oc get profiles.compliance.openshift.io -n openshift-compliance
``` 

This should list a number of available profiles, including the three "moderate" profiles listed at the top of this document.

## Scanning the OpenShift Cluster

Now that the Compliance Operator is installed, you need to create `ScanSettingBinding`s that will tell the operator to execute the 3 "moderate" scans on a specific schedule.  For simplity sake, we will bind these profiles to the "default" `ScanSetting`, which runs the scans as soon as it is create, and then each day at 1am GMT.

You can see these `ScanSettingBinding`s in the "manifests" directory of this repository.  To execute this from the CLI, simply run the following command:

```
oc apply -k https://github.com/pittar/ocp4-compliance-pbmm/manifests
```

This will kick off the inital scans of the cluster.  This can take some time as it will need to run scans on each cluster node. 

## Viewing Scan Results - Easy Way for Demos

The easiest way to view your scan results is to deploy the "compliance view" app that I created for this purpose.  This is a basic NGiNX image that mounts a shared PVC that will have report content.  There is also a `CronJob` that extracts the scan results and builds OpenSCAP HTML reports that are stored in the shared PVC.  This is a convenient way to display reports for demos.  The CronJob is set to run at 5am UTC every day, so the reports should continually be fresh.

To deploy the application and CronJob:

```
oc apply -k https://github.com/pittar/ocp4-compliance-pbmm/compliance-view/mainfests
```

Since the CronJob won't run until 5am UTC, you need to kick off the first run manually:

```
oc create job --from=cronjob/compile-compliance-report-cronjob initial-reports -n report-view
```

The CronJob will start by scaling the NGiNX Deployment to 0, then it will extract/generate reports (this take a few minutes), and finally scale the NGiNX deployment back to 1.  At this point, you can access the route and view the reports.

## Viewing Scan Results

Another way to view scan results is with the [oc compliance](https://docs.openshift.com/container-platform/4.9/security/compliance_operator/oc-compliance-plug-in-using.html) plugin.  If you are not using Red Hat Enterprise Linux 8, then you may have to build the plugin yourself.  This is pretty straight forward if you have golang installed on your machine (1.15+).

If you have golang installed and want to build the `oc-compliance` plugin yourself, simply clone the project and run make / make install.

```
git clone https://github.com/openshift/oc-compliance
cd oc-compliance
make
# you may need sudo if you have `oc` installed somewhere besides your home directory
make install
```

### Listing Scan Results

You can list the results of the scan, along with PASS/FAIL/MANUAL designation by running the command:

```
oc get compliancecheckresults -n openshift-compliance  
```

This will give you a long list of results.  If you want to see an individual result, pick one from the list, sush as `ocp4-moderate-node-worker-file-owner-kubelet-conf` and run:

```
oc compliance view-result ocp4-moderate-node-worker-file-owner-kubelet-conf -n openshift-compliance
```

You will see an output that looks like this:

```
+----------------------+---------------------------------------------------+
|         KEY          |                       VALUE                       |
+----------------------+---------------------------------------------------+
| Title                | Verify User Who Owns The                          |
|                      | Kubelet Configuration File                        |
+----------------------+---------------------------------------------------+
| Status               | PASS                                              |
+----------------------+---------------------------------------------------+
| Severity             | medium                                            |
+----------------------+---------------------------------------------------+
| Description          | To properly set the owner of                      |
|                      | /etc/kubernetes/kubelet.conf ,                    |
|                      | run the command:                                  |
|                      |                                                   |
|                      |                                                   |
|                      |                                                   |
|                      | $ sudo chown root                                 |
|                      | /etc/kubernetes/kubelet.conf                      |
+----------------------+---------------------------------------------------+
| Rationale            | The kubelet configuration file                    |
|                      | contains information about the                    |
|                      | configuration of the OpenShift                    |
|                      | node that is configured on                        |
|                      | the system. Protection of this                    |
|                      | file is critical for OpenShift                    |
|                      | security.                                         |
+----------------------+---------------------------------------------------+
| Instructions         | To check the ownership of                         |
|                      | /etc/kubernetes/kubelet.conf,                     |
|                      |                                                   |
|                      | you'll need to log into a node                    |
|                      | in the cluster.                                   |
|                      |                                                   |
|                      | As a user with administrator                      |
|                      | privileges, log into a node in                    |
|                      | the relevant pool:                                |
|                      |                                                   |
|                      |                                                   |
|                      |                                                   |
|                      | $ oc debug node/$NODE_NAME                        |
|                      |                                                   |
|                      |                                                   |
|                      |                                                   |
|                      | At the sh-4.4# prompt, run:                       |
|                      |                                                   |
|                      |                                                   |
|                      |                                                   |
|                      | # chroot /host                                    |
|                      |                                                   |
|                      |                                                   |
|                      |                                                   |
|                      |                                                   |
|                      |                                                   |
|                      | Then,run the command:                             |
|                      |                                                   |
|                      | $ ls -lL                                          |
|                      | /etc/kubernetes/kubelet.conf                      |
|                      |                                                   |
|                      | If properly configured, the                       |
|                      | output should indicate the                        |
|                      | following owner:                                  |
|                      |                                                   |
|                      | root                                              |
+----------------------+---------------------------------------------------+
| CIS-OCP Controls     | 4.1.6                                             |
+----------------------+---------------------------------------------------+
| NERC-CIP Controls    | CIP-003-8 R6, CIP-004-6 R3,                       |
|                      | CIP-007-3 R6.1                                    |
+----------------------+---------------------------------------------------+
| NIST-800-53 Controls | CM-6, CM-6(1)                                     |
+----------------------+---------------------------------------------------+
| Available Fix        | No                                                |
+----------------------+---------------------------------------------------+
| Result Object Name   | ocp4-moderate-node-worker-file-owner-kubelet-conf |
+----------------------+---------------------------------------------------+
| Rule Object Name     | ocp4-file-owner-kubelet-conf                      |
+----------------------+---------------------------------------------------+
| Remediation Created  | No                                                |
+----------------------+---------------------------------------------------+
```

If you'd like to export all of your results, youc an write a simple bash script such as:

```
#!/bin/bash

ccrs="$(oc get ccr -o custom-columns=NAME:.metadata.name --no-headers=true)"
for ccr in $ccrs
do
    echo "$(oc compliance view-result $ccrs -n openshift-compliance)"
    echo ""
done
```

Then run it and direct the output to a file.

```
./export-results.sh > full-results.txt
```

This might take a while to run, so you might want to keep an eye on the file size of your output file to make sure it continues growing.

## Generating HTML Reports

You can also download the raw results from the scan and generate an HTML report using the `oscap` cli.  For simplicity, I've created a simple helper container image that contains `oc`, `oc-compliance`, and `oscap` to demonstrate the process:

1. Start the tools container and enter a bash session:
```
docker run -it --rm -v <your local directory>:/data/output quay.io/pittar/oc-compliance:latest /bin/bash
```

2. Login to your OpenShift cluster (in the container):
```
oc login --token=<token> --server=<openshift api endpoint>
```

3. Fetch the raw compliance results:
```
oc compliance fetch-raw scansettingbindings ocp4-moderate-compliance -n openshift-compliance -o /data/output/
oc compliance fetch-raw scansettingbindings rhcos4-moderate-compliance -n openshift-compliance -o /data/output/
```

4. Build HTML reports based on the raw results.  They will be in /data/output/html:
```
build-reports.sh /data/output
```

When you are done, you should have a number of html reports in your local directory (in my case, `<your local directory>/html`.
