# OpenShift Virtualization Quick-Start

This is a quick start guide to get you started with OCP-V.

- [OpenShift Virtualization Quick-Start](#openshift-virtualization-quick-start)
  * [Installation](#installation)
    + [Online Clusters](#online-clusters)
    + [Disconnected Clusters](#disconnected-clusters)
  * [Getting your VM image into OCP-V](#getting-your-vm-image-into-ocp-v)
    + [Import image to a registry](#import-image-to-a-registry)
    + [Import image to a DataVolume](#import-image-to-a-datavolume)
  * [Deploy VM using a registry image](#deploy-vm-using-a-registry-image)
  * [Deploy VM using a cloned Data Volume](#deploy-vm-using-a-cloned-data-volume)
  * [Network Configuration using Nmstate](#network-configuration-using-nmstate)
    + [Cloud Init](#cloud-init)
    + [Notes](#notes)


## Installation

OpenShift Virtualization is installed via OperatorHub. To install OpenShift Virtualization you will need to have cluster admin permissions.

### Online Clusters

For clusters that have internet connection with unrestricted access to quay.io and registry.redhat.io follow the [standard installation guide](https://docs.openshift.com/container-platform/4.5/virt/install/installing-virt-web.html#virt-subscribing-to-the-catalog_installing-virt-web)

### Disconnected Clusters

For both disconnected clusters and clusters with a registry proxy, you will need create a catalog source. For disconnected clusters you will need to mirror the related images to your local registry as well. There are two ways to do this.

The first is using [OCP docs for catalog creation and mirroring](https://docs.openshift.com/container-platform/4.5/operators/olm-restricted-networks.html). The limitation of this method is that you have to mirror the entire catalogue which can take anywhere between 1 to 5 hours and take up 20-30gb space. Most of the operators in the catalogue cannot be used in disconnected clusters so its not an ideal solution.

The second method is using [the custom catalog creation tool](https://github.com/arvin-a/openshift-disconnected-operators). This tool will let you create a custom catalogue with only the operators you want. Full instructions are provided in the repo.

Once the catalogue is up navigate to OperatorHub and install OpenShift Virtualization [using the standard instructions](https://docs.openshift.com/container-platform/4.5/virt/install/installing-virt-web).

## Getting your VM image into OCP-V

OCP-V supports importing of QCOW2 and RAW images. For the purposes of this guide we can use the Fedora qcow2 image found here https://alt.fedoraproject.org/cloud/.  

Here are two ways to import images to OCP-V.

### Import image to a registry

You will need to have podman installed for this method. Create a Dockerfile with the contents below and copy the qcow image to the same folder.

``` Dockerfile
FROM kubevirt/container-disk-v1alpha
ADD Fedora-Cloud-Base-32-1.6.x86_64.qcow2 /disk
```

Build and push the Dockerfile. Substitute "MyRegistry:5000" with your registry and port.

``` Shell
podman build -t MyRegistry:5000/fedora:32.1.6 .
podman push MyRegistry:5000/fedora:32.1.6
```

The fedora image is now available to be pulled through your registry. This will be demonstrated in following sections.

### Import image to a DataVolume

You can use the virtctl cli tool to import qcow images to a data volume.

You can install virtctl via the instruction in [the OCP docs](https://docs.openshift.com/container-platform/4.5/virt/install/) or get the latest version from [the project's GitHub site](https://github.com/kubevirt/kubevirt/releases).

This method requires a Storage Class to be setup so we can provision Persistent Volumes. If your Storage Class requires manual provisioning of PVs, create a PV with a storage size of 20GB.

Run the following command to upload the image to a Data Volume managed Persistent Volume

``` Shell
oc new-project vm-project
oc project vm-project
virtctl image-upload dv fedora-32-dv --size=20G --storage-class=local-storage --image-path=vm-images/Fedora-Cloud-Base-32-1.6.x86_64.qcow2
```

Once complete a Data Volume with the qcow image will be ready for use by a VM.

## Deploy VM using a registry image

If you want to create immutable VMs that have ephemeral or persistent storage, creating VMs from registry image is the easiest way to achieve this. Earlier we created registry image using the Fedora qcow. Use the sample yaml [fedora-immutable-vm-pod-network.yaml](https://github.com/arvin-a/OpenShift-Virtualization-Quick-Start/blob/master/samples/fedora-immutable-vm-pod-network.yaml) from this repo to create your fedora immutable VM.

Relevant section

``` yaml
      volumes:
        - containerDisk:
            image: 'MyRegistry:5000/fedora:32.1.6'
          name: rootdisk
```

## Deploy VM using a cloned Data Volume

It's equally easy to to create a traditional VM with persistent storage. Earlier we used virtctl cli to create a data volume from the Fedora qcow image. We can now use that data volume as the source of the clone for our VM instance. See the sample yaml  [fedora-vm-pod-network.yaml ](https://github.com/arvin-a/OpenShift-Virtualization-Quick-Start/blob/master/samples/fedora-immutable-vm-pod-network.yaml)

Relevant section.

``` yaml
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1alpha1
      kind: DataVolume
      metadata:
        name: fedora2-disk-0
      spec:
        pvc:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 120G
          storageClassName: local-storage
          volumeName: fedora2-pv
          volumeMode: Filesystem
        source:
          pvc:
            name: fedora-32-dv
            namespace: vm-project
```


## Network Configuration using Nmstate

Each VM is controlled via a virt-launcher pod that is created with each VM.
The default networking type for OCP-V VMs is Masquerade. The VM will be assigned a none routable IP and you will access the VM using the IP of the virt-launcher pod that was deployed alongside it.

``` yaml
          interfaces:
            - masquerade: {}
              model: virtio
              name: nic0
```

You can also connect to the host network by creating a bridge using Nmstate. Nmstate operator is installed with OVP-V and you can use NodeNetworkConfigurationPolicy (NNCP) object to update the host network settings.

Here is a sample config to create a bridge called br1 from an interface called eth1.

```yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1
spec:
  nodeSelector: 
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge using eth1 device
        type: linux-bridge
        state: up
        ipv4:
          dhcp: true
          enabled: true
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eth1
```

Once this is applied to your cluster use the following command to see the status of configuration update.

```shell
oc get nncp
oc get nnce
```

NNCP is a cluster wide configuration. For each namespace you have to create a Network Attachment Definition (NAD) to use the bride in your VMs. Since the configuration is a json embeded in a yaml, it is easier to create your initial NAD through the web console then use that yaml in your future automation.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br1
  name: br1
  namespace: vm-project
spec:
  config: >-
    {"name":"br1","cniVersion":"0.3.1","plugins":[{"type":"cnv-bridge","bridge":"br1","ipam":{}},{"type":"cnv-tuning"}]}
```

After applying the NAD yaml, you can now use the bridge in your VM. See relevant sections from a VM yaml below.

```yaml
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - bridge: {}
              model: virtio
              name: nic-0
      networks:
        - multus:
            networkName: br1
          name: nic-0
```


See [Openshift docs](https://docs.openshift.com/container-platform/4.5/virt/node_network/virt-updating-node-network-config.html) and [NMstate docs](https://www.nmstate.io/) for more details.



### Cloud Init

You can create initial VM configuration through cloud init. In the simple example below we enable password authentication and change the root password

```yaml
      volumes:
        - cloudInitNoCloud:
            userData: |
              #cloud-config
              ssh_pwauth: True
              chpasswd:
                list: |
                   root:password
                expire: False
              hostname: fedora1
          name: cloudinitdisk

```

This data can be hard coded like the example above or you can create a secret with that data and mount the secret instead (recommended).

```yaml
      volumes:
        - cloudInitConfigDrive:
            secretRef:
              name: vminit
          name: cloudinitdisk
```


### Notes

This document is a work in progress.
