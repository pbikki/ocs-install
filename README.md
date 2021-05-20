# OpenShift Container Storage on managed IBM Cloud OpenShift Cluster on VPC infra

Red Hat OpenShift Container Storage (OCS) is a highly integrated collection of cloud storage and data services for Red Hat OpenShift Container Platform. Follow through [IBM Cloud docs to install OCS](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-prep) add-on to ROKS cluster 

> Below is a gist from the cloud docs and may not cover complete configuration. This is will help install basic OCS cluster. For details, refer the above cloud docs. 

> Note the steps are only applicable to Managed OpenShift clusters (ROKS) on VPC infra on IBM Cloud

### Preqrequisites

- ROKS VPC cluster with minimum 3 16x64 workers (For HA, create workers in 3 different AZs)

### Install OCS

- (Optional) Sets COS to be default backing storage for noobaa. Details [Setting up an IBM Cloud Object Storage service instance](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-install#ocs-create-cos)
    - Setup COS instance
    - Create HMAC creds
    - Create `openshift-storage` NS, [namespace yaml](./os-namespace.yaml) 
    - Create `ibm-cloud-cos-creds` COS secret

- Enable OCS add-on. Wait until add-on is enabled and Ready
    ```
    $ ibmcloud oc cluster addon enable openshift-container-storage -c <cluster-name>

    $ ibmcloud oc cluster addon ls -c <cluster-name>
    OK
    Name                          Version   Health State   Health Status   
    openshift-container-storage   4.6.0     normal         Addon Ready (H1500)   
    vpc-block-csi-driver          3.0.0     normal         Addon Ready (H1500)   

    $ oc get pods -A | grep ibm-ocs-operator-controller-manager
    kube-system                                        ibm-ocs-operator-controller-manager-55667f4d68-7qfd5              1/1     Running     1          29h
    ```

- Create [custom SC](./custom-sc.yaml) (can copy the yaml of one of the existing SCs. The name of the SC should be less than 63 chars). Ref [IBM Cloud docs - OCS limiation](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-cluster-setup#ocs-limitations)

    For eg:  `ibmc-vpc-block-metro-retain-10iops-tier`  cannot be directly used due to limitation with character length. Instead, create another SC with same spec but a shorter name
    > For MZR clusters, only use StorageClasses with `volumeBindingMode: WaitForFirstConsumer` 

   ```
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: ibm-block-custom-ocs
   parameters:
     billingType: hourly
     classVersion: "1"
     csi.storage.k8s.io/fstype: ext4
     encrypted: "false"
     encryptionKey: ""
     profile: 10iops-tier
     region: ""
     resourceGroup: ""
     sizeRange: '[10-2000]GiB'
     tags: ""
     zone: ""
   provisioner: vpc.block.csi.ibm.io
   reclaimPolicy: Retain
   volumeBindingMode: WaitForFirstConsumer
  ```

- Prepare the `OcsCluster` [CR yaml](./ocs-cluster.yaml) and Create storage cluster

    Example yaml
    ```
    apiVersion: ocs.ibm.io/v1
    kind: OcsCluster
    metadata:
     name: ocscluster-vpc
    spec:
     monStorageClassName: ibm-block-custom-ocs # For multizone clusters, specify a 'metro' storage class
     monSize: 20Gi
     osdStorageClassName: ibm-block-custom-ocs # For multizone clusters, specify a 'metro' storage class
     osdSize: 150Gi # The OSD size is the total storage capacity of your OCS storage cluster
     numOfOsd: 3
     billingType: hourly
     ocsUpgrade: false
    ```
    ```
    ▶ oc create -f ocs-cluster.yaml 
    ocscluster.ocs.ibm.io/ocscluster-vpc created
    ```

### References
- [IBM Cloud docs - OpenShift Container Storage](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-prep)
- [IBM Cloud docs - Managing OCS deployment](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-manage-deployment)
- [Introduction to Red Hat OpenShift Container Storage](https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.6/html/planning_your_deployment/introduction-to-openshift-container-storage-4_rhocs)
