# How to handle issues on nodes for Openshift Virtualization and the impact on its Virtual Machines

## Node Health Check

  * The Node Health Check Operator is an Red Hat OpenShift add-on operator which implements a failure detection system that monitors node conditions. 
  * It does not have a built-in fencing or remediation system and so must be configured with an external system that provides such features. 
  * By default, it is configured to utilize the Self Node Remediation system.

