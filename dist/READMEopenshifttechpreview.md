# OVN tech preview on OpenShift/origin/kubernetes

NOTE:
- This is a temporary approach to working with the tech preview. Ultimately
4.0 will install using the network operator.

You can preview ovn overlay networks on an OpenShift 4.0 cluster by replacing
the installed CNI plugin with OVN.
A v3.11 cluster installs with no plugin and 4.0 (currently) installs flannel.

NOTES:
- This is Tech Preview, not Production, and it will change over time.
- There is no upgrade path. New install only.
- This is limited to a single master in the cluster (no high availability).


## Installation:

Install OKD-v4.0 as instructed in the OKD documentation. More TBD.


Install OKD-v3.11 as instructed in the OKD documentation. Make the following change
before running the cluster install playbook:
1. In the ansible host file for the cluster change:
```
os_sdn_network_plugin_name='redhat/openshift-ovs-multitenant'
```
  to
```
os_sdn_network_plugin_name='cni'
openshift_use_openshift_sdn='False'
```

When the cluster install is complete and the cluster is up there will
be no cluster networking.


Clone ovn-kubernetes on a convenient host where you can run ansible-playbook:
```
# git clone https://github.com/openvswitch/ovn-kubernetes
# cd ovn-kubernetes/dist/ansible
```

Edit the hosts file adding the name of the cluster master and select the ovn_image
and ovn_image_pull_policy. The default is the community image.

Provision the cluster for OVN:
```
# ./run-playbook
```

OVN may be removed from the cluster by running:
```
# ./run-playbook uninstall
```

The yaml/{ovnkube.yaml,ovnkube-master.yaml} files are now created as follows:
```
# cd ../images
# make daemonsetyaml
```
The daemonsets are now in template files that are expanded on the master. The
previous daemonsets in dist/yaml have been deleted and can be reconstructed 
using the above make.

```
# oc project
Using project "ovn-kubernetes" on server "https://wsfd-netdev22.ntdv.lab.eng.bos.redhat.com:8443".
# oc get nodes
NAME                                        STATUS    ROLES           AGE       VERSION
wsfd-netdev22.ntdv.lab.eng.bos.redhat.com   Ready     infra,master    43d       v1.10.0+b81c8f8
wsfd-netdev28.ntdv.lab.eng.bos.redhat.com   Ready     compute         43d       v1.10.0+b81c8f8
wsfd-netdev35.ntdv.lab.eng.bos.redhat.com   Ready     compute         43d       v1.10.0+b81c8f8
# oc get ds
NAME             DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR                         AGE
ovnkube          3         3         3         3            3           beta.kubernetes.io/os=linux                                       1h
ovnkube-master   1         1         1         1            1           beta.kubernetes.io/os=linux,node-role.kubernetes.io/master=true   1h

# oc get po
NAME                   READY     STATUS    RESTARTS   AGE
ovnkube-cggxz          3/3       Running   0          1h
ovnkube-ltpzr          3/3       Running   0          1h
ovnkube-master-mj55x   2/2       Running   0          1h
ovnkube-pdw7h          3/3       Running   0          1h
```

At this point ovn is providing networking for the cluster.

## Images:

There is a single docker image that is used in all of the ovn daemonsets.
The desired image is entered in the hosts file.

Images can be found in docker.io, one of the official OKD repositories or
a user provided image repository.

A development image can be built in the openvswitch/ovn-kubernetes git repo
and taged and pushed to a private image repo.

The OCP image is built in the openshift/ose-ovn-kubernetes repo from rhel:7
with openvswitch from the fastdatapath repo.
The default community image is built from centos:7 with openvswitch from
http://cbs.centos.org/kojifiles/packages/.  The image can also
be built from fedora:28 with openvswitch from fedora.

The OCP image is available in the following:
```
registry.access.redhat.com/
brew-pulp-docker01.web.prod.ext.phx2.redhat.com:8888/
aws.openshift.com:443/
```
The OKD 4.0 image name includes the build tag:
```
openshift3/ose-ovn-kubernetes:v4.0
```

The community image based on current development is:
```
docker.io/ovnkube/ovn-daemonset:latest
```
The the community image is the default.


Alternatively, the image can be built from the ovn-kubernetes repo.
When doing this edit the Makefile to tag and push the image to your existing
docker registry.  Edit the hosts file to reference the image.

1. build the ovn binaries, copy them to the Dockerfile directory and build the image,
tag and push it to your registry:
```
$ cd ovn-kubernetes/dist/images
$ make
```

or

```
$ make fedora
```

In a development cycle, a new image can be built and pushed and the ovnkube-master and ovnkube daemonsets
can be deleted and recreated.

```
# cd ovn-kubernetes/dist/yaml
# oc project ovn-kubernetes
# oc delete -f ovnkube.yaml
# oc delete -f ovnkube-master.yaml
# oc create -f ovnkube-master.yaml
# oc create -f ovnkube.yaml
```
