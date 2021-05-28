# OpenShift Container Storage on managed IBM Cloud OpenShift Cluster on VPC infra

Red Hat OpenShift Container Storage (OCS) is a highly integrated collection of cloud storage and data services for Red Hat OpenShift Container Platform. Follow through [IBM Cloud docs to install OCS](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-prep) add-on to ROKS cluster 

OCS storage cluster has 3 deployment approaches depending on how storage will be made available to the cluster:

- **Internal - dynamic storage** : OCS dynamically provisions storage using the cloud provider's Storage class and sizes that are specified
- **Internal - attached local devices**: In this approach, set of local disks are available on the nodes and local storage operator (LSO) is installed to recognize the disks and make them ready for OCS deployment
- **External** - This approach allows OCS to expose the Red Hat Ceph Storage services running outside of the OpenShift cluster as storage classes

Read more about [storage deployment approaches](https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.6/html/planning_your_deployment/ocs-architecture_rhocs#storage-cluster-deployment-approaches_rhocs)

> :exclamation: IBM Cloud OCS add-on for VPC clusters,as of now, only supports `Internal - dynamic storage` mode which is a straight-forward approach to setup an OCS cluster. 

> `Internal - attached local devices` is supported on classic infra but not VPC as SDS node flavors are only available on classic infra as of now. Refer [Software-defined storage (SDS) machines](https://cloud.ibm.com/docs/openshift?topic=openshift-planning_worker_nodes#sds)


### Preqrequisites

- Install the [IBM Cloud CLI](https://cloud.ibm.com/docs/cli?topic=cli-install-ibmcloud-cli) and the [Red Hat OpenShift on IBM Cloud plug-in](https://cloud.ibm.com/docs/openshift?topic=openshift-openshift-cli#cs_cli_install_steps)
- [Installing the OpenShift Origin CLI (oc)](https://cloud.ibm.com/docs/openshift?topic=openshift-openshift-cli#cli_oc)
- ROKS VPC cluster with minimum 3 16x64 workers (For HA, create workers in 3 different AZs). [Create managed OpenShift cluster on VPC infra](https://cloud.ibm.com/docs/openshift?topic=openshift-clusters#clusters_vpcg2)
- [Login to the openshift cluster](https://cloud.ibm.com/docs/openshift?topic=openshift-access_cluster#access_public_se)


> Steps in [Install OCS](#install-ocs) section is a gist from the cloud docs and may not cover complete configuration. This is will help install basic OCS cluster. For details, refer the [cloud docs](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-prep)

> :speaker: Note the steps are only applicable to Managed OpenShift clusters (ROKS) on VPC infra on IBM Cloud

### Install OCS

- (Optional) Sets Cloud Object Storage(COS) to be default backing storage for noobaa. 
    - Setup COS instance
    - Create HMAC creds for COS
    - Create `openshift-storage` NS, [namespace yaml](./os-namespace.yaml) 
    - Create `ibm-cloud-cos-creds` COS secret (secret name should be the same, OCS will look for this secret)
  
   Refer [Setting up an IBM Cloud Object Storage service instance](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-install#ocs-create-cos) for details and commands to complete the above steps using IBM Cloud CLI 

- Enable OCS add-on. Wait until add-on is enabled and Ready. Steps below show enabling the add-on using IBM Cloud CLI. IT can also be [enabled through IBM Cloud console](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-install#install-ocs-console)
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

- Create [custom StorageClass](./custom-sc.yaml) (can copy the yaml of one of the existing SCs. The name of the SC should be less than 63 chars). Refer [IBM Cloud docs - OCS limitations](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-cluster-setup#ocs-limitations)
   To list existing SCs on the cluster,
   ```
   $ oc get sc
   ```
   > One of the IBM block storage classes can be used. For MZR clusters, it is recommended to StorageClasses with `volumeBindingMode: WaitForFirstConsumer` 
   
    For eg:  `ibmc-vpc-block-metro-retain-10iops-tier`  cannot be directly used due to limitation with character length. Instead, create another SC with same spec but a shorter name
  

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
  ```
  $ oc create -f custom-oc.yaml
  ```

- Prepare the `OcsCluster` [CustomResource yaml](./ocs-cluster.yaml) and Create storage cluster

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
    > The above example installs OCS componenets on all nodes of the cluster. To install on specific node, refer [Creating your OCS storage cluster CRD](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-cluster-setup#ocs-vpc-deploy-crd)

### References
- [IBM Cloud docs - OpenShift Container Storage](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-storage-prep)
- [IBM Cloud docs - Managing OCS deployment](https://cloud.ibm.com/docs/openshift?topic=openshift-ocs-manage-deployment)
- [Introduction to Red Hat OpenShift Container Storage](https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.6/html/planning_your_deployment/introduction-to-openshift-container-storage-4_rhocs)
