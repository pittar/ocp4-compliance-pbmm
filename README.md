# OpenShift Compliance Operator - Mapping to ITSG-33

THe OpenShift [Compliance Operator](https://docs.openshift.com/container-platform/4.9/security/compliance_operator/compliance-operator-supported-profiles.html) offers a number of scanning profiles.  The ones that are most relevant for Canadian Government departments that need to prove compliance of their platforms are:

* **ocp4-moderate** - NIST 800-53 Moderate-Impact Baseline for Red Hat OpenShift - Platform level
* **ocp4-moderate-node** - NIST 800-53 Moderate-Impact Baseline for Red Hat OpenShift - Node level
* **rhcos4-moderate** - NIST 800-53 Moderate-Impact Baseline for Red Hat Enterprise Linux CoreOS

You can see the [NIST-800-53 controls here](https://csrc.nist.gov/Projects/risk-management/sp800-53-controls/release-search#!/800-53).

The reason these scans are relevant is that the [ITSG-33](https://cyber.gc.ca/en/guidance/it-security-risk-management-lifecycle-approach-itsg-33) standard maps almost 1:1 to NIST-800-53.  For this reason, running the above scans on your OpenShift environment can give greatly improve your ability to remediate issues and provide an evidence package to support your authority to operate.

## Installing the Compliance Operator

The Compliance Operator can be installed in a few different ways:

1. Through OperatorHub: An OpenShift cluster administrator can login to OpenShift, search the integrated "OperatorHub" for "Compliance Operator", then accept all defaults to install the Compliance Operator into the recommended namespace.
2.[Red Hat Advanced Cluster Management for Kubernetes](https://www.redhat.com/en/technologies/management/advanced-cluster-management) can be used to create a "Policy" that automatically installs the Compliance Operator on all managed OpenShift clusters.
3. GitOps: A GitOps approach can be taken in order to use a tool such as [Argo CD (OpenShift GitOps)](https://argo-cd.readthedocs.io/en/stable/) to install the Compliance Operator.

Going through each option in detail is out of scope for this document.  For now, you can either take the 1st approach and install the Compliance Operator through the OpenShift Admin UI, or, if you are already logged in to your cluster as a `cluster-admin`, you can execute the following command to install the Compliance Operator with default settings.

```
oc apply -k https://github.com/redhat-cop/gitops-catalog/compliance-operator/operator/overlays/release-0.1
```

After a few minutes, the Compliance Operator will be running in a new namespace called `openshift-compliance`.

You will know when the operator is ready as you will now be able to view the different profiles that are available:

```
oc get profiles.compliance -n openshift-compliance
``` 

This should list a number of available profiles, including the three "moderate" profiles listed at the top of this document.

## Scanning the OpenShift Cluster

Now that the Compliance Operator is installed, you need to create a `ScanSettingBinding` that will tell the operator to execute the 3 "moderate" scans on a specific schedule.  For simplity sake, we will bind these profiles to the "default" `ScanSetting`, which runs the scans as soon as it is create, and then each day at 1am GMT.

You can see this `ScanSettingBinding` in the "manifests" directory of this repository.  To execute this from the CLI, simply run the following command:

```
oc apply -k https://github.com/pittar/ocp4-compliance-pbmm/manifests
```

This will kick off the inital scans of the cluster.  This can take some time as it will need to run scans on each cluster node. 
