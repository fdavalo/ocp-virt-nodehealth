# How to handle issues on nodes for Openshift Virtualization and the impact on its Virtual Machines

Read 
  * https://access.redhat.com/documentation/en-us/workload_availability_for_red_hat_openshift/23.2/html-single/remediation_fencing_and_maintenance/index?extIdCarryOver=true&sc_cid=701f2000001Css5AAC#about-remediation-fencing-maintenance
  * https://www.medik8s.io/
  * https://www.redhat.com/fr/blog/keeping-virtual-machines-available-by-allowing-nodes-to-self-repair?channel=/fr/blog/channel/red-hat-openshift

## Node Health Check

  * The Node Health Check Operator is an Red Hat OpenShift add-on operator which implements a failure detection system that monitors node conditions. 
  * It does not have a built-in fencing or remediation system and so must be configured with an external system that provides such features. 
  * By default, it is configured to utilize the Self Node Remediation system.


         apiVersion: remediation.medik8s.io/v1alpha1
         kind: NodeHealthCheck
         metadata:
           name: nhc-worker-default
         spec:
           minHealthy: 51%
           remediationTemplate:
             apiVersion: self-node-remediation.medik8s.io/v1alpha1
             kind: SelfNodeRemediationTemplate
             name: self-node-remediation-resource-deletion-template
             namespace: openshift-operators
           selector:
             matchExpressions:
               - key: node-role.kubernetes.io/worker
                 operator: Exists
           unhealthyConditions:
             - duration: 30s
               status: 'False'
               type: Ready
             - duration: 30s
               status: Unknown
               type: Ready
               
## Self Node Remediation

  * The Self Node Remediation Operator is an Red Hat OpenShift add-on operator which implements an external system of fencing and remediation that reboots unhealthy nodes and deletes resources, such as, Pods and VolumeAttachments. 
  * The reboot ensures that the workloads are fenced, and the resource deletion accelerates the rescheduling of affected workloads. 
  * Unlike other external systems, Self Node Remediation does not require any management interface, like, for example, Intelligent Platform Management Interface (IPMI) or an API for node provisioning.

  * Self Node Remediation can be used by failure detection systems, like Machine Health Check or Node Health Check.

First install the Operator and specify a SelfNodeRemediationTemplate with the appropriate remediation type (resourcedeletetion is a good default)

     apiVersion: self-node-remediation.medik8s.io/v1alpha1
     kind: SelfNodeRemediationTemplate
     metadata:
       name: self-node-remediation-resource-deletion-template
       namespace: openshift-operators
     spec:
       template:
         spec:
           remediationStrategy: ResourceDeletion
      
ResourceDeletion
  * This remediation strategy removes the pods and associated volume attachments on the node, rather than the removal of the node object. 
  * This strategy recovers workloads faster. ResourceDeletion is the default remediation strategy.

You can tune configuration in the SelfNodeRemdiationConfig, but do not forget : 

       apiVersion: self-node-remediation.medik8s.io/v1alpha1
       kind: SelfNodeRemediationConfig
       metadata:
         name: self-node-remediation-config
         namespace: openshift-operators
       spec:
         apiServerTimeout: 5s
         peerApiServerTimeout: 5s
         isSoftwareRebootEnabled: true
         watchdogFilePath: /dev/watchdog
         peerDialTimeout: 5s
         peerUpdateInterval: 15m
         apiCheckInterval: 15s
         peerRequestTimeout: 5s
         safeTimeToAssumeNodeRebootedSeconds: 180
         maxApiErrorThreshold: 3
  

  * If the node is not down, a reboot will be started by a daemonset/pod on the node
  * You won't be allowed to go under around 3 minutes of waiting if the node is not rebooting
  * After this verification, the remediation process will started :
      * remove volume attachment which will enable pod deletion (usually stuck in Terminating state when there are persistent volumes mounted (volumeattachment))
  
