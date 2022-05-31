# How to Setup Post-Install Cluster Configurations using GitOps
This document will walk you through the process of creating post installation configuration using git and Openshift GitOps.  

The configurations are stored in the `ocp-gitops-infra-config` repo and are referenced by the OpenShift GitOps ArgoCD application 
which is running in the hub cluster.  

The configuration is based on the Kustomization framework to allow reuse and patching of yaml manifests files. 
There is a playbook that will generate a configuration for a cluster based on a variables yaml file.  Once the 
configuration is in place it can be altered and maintained in the git repository and updates are applied to the cluster 
using the OpenShift GitOps ArgoCD manager.

## Openshift GitOps Console
The console for OpenShift GitOps can be accessed from the link below.

> NOTE: Applications and Sync Status can be viewed from the Gitops console but applications need to be deployed using the Playbook.

https://openshift-gitops-server-openshift-gitops.apps.ocp-dev-hub.\<domain\>/

## How to Configure Cluster using OpenShift GitOps

1. Create the [configuration variable](#configuring-cluster-variables) file in the ocp-infra-deploy repository.

2. Run the `configure-gitops-infra` playbook using the following example. The playbooks reside in the `ocp-infra-lib` repository.  

    ```
    $ ansible-playbook playbooks/configure-gitops-infra.yaml \
      -i ../ocp-infra-deploy/<env>/inventory \
      -e state=present \
      -e hub_cluster_name=<hub-cluster-name> \
      -e cluster_name=<cluster-name> \
      -e aws_region=<aws-region> \
      --vault-password-file=~/.passfile
    ```
    > **Notes:**
    >
    > *Dependencies*
    >
    > The `ocp-infra-deploy` repository must be checked out in the same directory as `ocp-infra-lib` in order to access the required inventory file, and `ocp-gitops-infra-config` must be checked out in the same location as well.
    >
    > *Variables*
    >
    > `env` The environment for your desired inventory (dev, evt)
    >
    > `hub_cluster_name` The name of the hub cluster that will manage this cluster
    >
    > `cluster_name` This must match the name of the cluster created using RHACM
    >
    > `aws_region` The AWS region the cluster will be deployed to. (Update as desired)
    >
    > *Ansible Vault*
    >
    > `--vault-password-file` This argument will direct Ansible Vault to a password file in order to access encrypted values 
    > for the install. This file should always be `~/.passfile`. Follow the [instructions above](#ansible-vault-password-file) to create the passfile.

3. Commit the generated cluster configuration files to the `ocp-gitops-infra-config` repository using the example below.
    ```
    $ cd ../ocp-gitops-infra-config
    $ git add clusters/<new-cluster>
    $ git commit -m "Added new cluster configuration"
    $ git push origin     
    ```
4. Run the `playbooks/configure-infra-argo` playbook using the following example. 
    > NOTE: The cluster name must match the name of cluster created and the target_branch must be the branch commited in step 3.
    ```
    $ ansible-playbook playbooks/configure-infra-argo.yaml  \
      -i ../ocp-infra-deploy/<env>/inventory \
      -e state=present \
      -e hub_cluster_name=<hub-cluster-name> \
      -e cluster_name=<cluster-name> \
      -e aws_region=<aws-region> \
      -e target_branch='<target-branch>' \
      --vault-password-file=~/.passfile
    ```

5. Verify the application is created and the Sync status is valid using the Openshift [Openshift GitOps Console](https://openshift-gitops-server-openshift-gitops.apps.ocp-hub.<domain>/)
> NOTE: The password for the console can be found in the hub cluster in the openshift-gitops secret named `openshift-gitops-cluster`. The username is `admin`.


## Configuring Cluster Variables

Create a variables file in the `ocp-infra-deploy` folder under the path `./<cluster-name>/infra_vars.yaml`

* Use "machinesets" to create infra, storage and additional worker nodes.
* Use "installed_workers" to add labels to worker nodes created during install.
* Use "registry" variable to configure additional registries
* Use "environments" and "additional_manifests" to reference existing manifests that are in the ocp-gitops-infra-config repository.

```
# infra_vars.yaml 
#
# Additional MachineSets can be added here.  Additional labels are added to the nodes when defined.  
#
machinesets:
  - name: infra
    role: infra
    ## For HA you can create multiple sets accross zones
    availability_zones:
      - name: "{{ aws_region }}a"
        replicas: 1
      - name: "{{ aws_region }}b"
        replicas: 1
      - name: "{{ aws_region }}c"
        replicas: 1
    provider_region: "{{ aws_region }}"
    instance_type: "m5.2xlarge"
  - name: ocp-worker
    role: worker
    availability_zones:
      - name: "{{ aws_region }}a"
        replicas: 1
      - name: "{{ aws_region }}b"
        replicas: 1
    provider_region: "{{ aws_region }}"
    instance_type: "m5.xlarge"
    ## Labels for pod scheduling can be added
  - name: gpu
    role: worker
    availability_zones:
      - name: "{{ aws_region }}a"
        replicas: 1
    provider_region: "{{ aws_region }}"
    instance_type: "p3.2xlarge"
    no_taints: 'true'
  - name: storage
    role: infra
    enabled: true
    availability_zones:
      - name: "{{ aws_region }}a"
        replicas: 1
      - name: "{{ aws_region }}b"
        replicas: 1
      - name: "{{ aws_region }}c"
        replicas: 1
    provider_region: "{{ aws_region }}"
    instance_type: "m5.4xlarge"
    additional_labels:
      cluster.ocs.openshift.io/openshift-storage: ""
    ## Declare Taints for the nodes as a list
    taints:
      - effect: NoSchedule
        key: node.ocs.openshift.io/storage
        value: 'true'

## Autoscale Support
machine_autoscale:
  enabled: 'true'
  autoscale_max_nodes: 31
  autoscale_max_memory: 900
  autoscale_min_memory: 512
  autoscale_min_cores: 128
  autoscale_max_cores: 256
  machinesets:
    - zone: "{{ aws_region }}a"
      name: ocp-worker
      min_replicas: 1
      max_replicas: 6
    - zone: "{{ aws_region }}b"
      name: ocp-worker
      min_replicas: 1
      max_replicas: 6
    - zone: "{{ aws_region }}a"
      name: gpu
      min_replicas: 0
      max_replicas: 6

#
## To configure labels on existing workers configure the following variables
#
installed_workers:
  additional_labels:
    ocp: "v1"

#
## To configure registry options for only allowing 'gci.io' and redhat registries use the 
#
registry:
  secure_registries:
    - registry.redhat.io
    - registry.access.redhat.com
    - registry.connect.redhat.com
    - nvcr.io
    - quay.io
    - gcr.io
    - docker.io
    - image-registry.openshift-image-registry.svc:5000
  insecure_registries:
    - example.com
  
  internal_registry_hostname:
    - image-registry.openshift-image-registry.svc:5000

#
## To include additional pre-defined kustomization operators define the list below
#
operators:
  - elasticsearch-operator
  - openshift-logging-operator
  - ocs-operator
  - prometheus-grafana-operator

#
## To include additional pre-defined kustomization environments define the list below
#
environments:
  - dev
  - infra-nodes
  - openshift-logging

#
## To include the observability stack include the section below
#
observability:
  - all

#
## To include additional pre-defined kustomization manifests folders define the list below
#
additional_manifests:
  - openshift-storage
```

# How to Configure GPU Compute Nodes

1. Create the gpu machinesets by adding a gpu machineset definition to the [configuration variables](#configuring-cluster-variables) 
file in the `ocp-infra-deploy` repository for the target cluster and running the `configure-gitops-infra.yaml` playbook.

2. Commit the machineset configuration to the gitops repository if using the ArgoCD to maintain the cluster or apply 
the new machine sets to the target cluster.

3. Enable Red Hat subscriptions entitlements by running the `configure-entitlements` playbook.
    > NOTE: The \<TODO\> entitlements can be cloned from the variables file located in the ocp-infra-deploy/ocp-dev repository.
    ```
    $ ansible-playbook ./playbooks/configure-cluster-entitlements.yaml \
      -i ../ocp-infra-deploy/<env>/inventory \
      -e state=present  \
      -e @../ocp-infra-deploy/<your cluster>/rh_entitlement.yaml \
      -e hub_cluster_name=<hub-cluster-name> \
      -e cluster_name=<cluster-name> \
      -e aws_region=<aws-region> \
      --vault-password-file=~/.passfile
    ```
    > NOTE: The Red Hat subscription entitlement pem file must be defined in a variables file  `../ocp-infra-deploy/<cluster-name>/rh_entitlement.yaml`.

4. Configure GPU manifests in the cluster
 
    > Add the policy manifest to `ocp-gitops-infra-config/<cluster-name>/kustomization.yaml`
    ```
    ...
    bases:
    - ../../manifests/nvidia-gpu-policy
    ...
    ```
   
    > Add the operator manifest to `ocp-gitops-infra-config/<cluster-name>/operators/kustomization.yaml`
   ```
   ...
   bases:
   - ../../../manifests/nvidia-gpu-operator
   ...
   ```
    
5. Verify the gpu resources get created by verifying DaemonSets in the gpu-operator-resources namespace are created with 
the pod count matching the number of GPU nodes defined.

6. Once the pods are running you can verify GPU is functional by running the sample pod shown below. 
    ```
    oc create -f - <<EOF
    apiVersion: v1
    kind: Pod
    metadata:
      name: nvidia-smi
    spec:
      restartPolicy: Never
      containers:
      - image: nvidia/cuda:11.4.0-runtime-ubi8 
        name: nvidia-smi
        command: [ nvidia-smi ]
        resources:
          limits:
            nvidia.com/gpu: 1
          requests:
            nvidia.com/gpu: 1
    EOF
    ```
    Verify logs output once the pods has completed.
    ```
    oc logs nvidia-smi
    ```

