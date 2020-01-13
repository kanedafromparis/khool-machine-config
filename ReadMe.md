# Openshift MachineConfig

## Introduction

This is a sample tutorial to test and apply [MachineConfig](https://docs.openshift.com/container-platform/4.2/architecture/architecture-rhcos.html). It is motivated by 2 issues:

- ntp configuration
 in order to use internal [Amazon ntp server](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html)

- crio configuration
 in order to mitigate the [cve-2019-14891](https://access.redhat.com/security/cve/cve-2019-14891)

## Notice

This sample tutorial is based on Openshift 4.2 (4.2.0) on AWS, It should works in other enviroment, but devil reside in thoses king of details ;-)

## Prepare the cluster

In order to secure the operation, we will create a temporary machine playground

### MachineSet

[MachineSet](https://docs.openshift.com/container-platform/4.2/machine_management/creating_machinesets/creating-machineset-aws.html) are openshift abstraction of your machines, that will be workers node.

you can access it :

on the console in https://console-openshift-console.apps.myopenshiftcluster.com/k8s/ns/openshift-machine-api/machine.openshift.io~v1beta1~MachineSet

with openshift cli you should obtain something like this :

```bash
oc get -n openshift-machine-api machineset
NAME                                       DESIRED   CURRENT   READY   AVAILABLE   AGE
myopenshiftcluster-worker-eu-central-1a   1         1         1       1           6d4h
myopenshiftcluster-worker-eu-central-1b   1         1         1       1           6d4h
myopenshiftcluster-worker-eu-central-1c   1         1         1       1           6d4h
```

central-1{a,b,c} are the amazon zone hosting my ec2 nodes.
_Notice_ : it can be scale  with oc scale like (```oc scale -n openshift-machine-api machineset myopenshiftcluster-worker-eu-central-1c --replicas=2```)

So we will create a new MachineSet that will be used for our MachineConfig testing.

the easier way to do so is to copy an existing machineset.

```bash
oc get -n openshift-machine-api machineset myopenshiftcluster-worker-eu-central-1c --export -o json | jq '.metadata.labels.playground += "OK" | .metadata.name = "my-test-playground" | del(.metadata.selfLink) | .spec.template.metadata.labels["machine.openshift.io/cluster-api-machine-role"]="my-test-playground" | .spec.template.spec.metadata.labels["node-role.kubernetes.io/worker"] += "my-test-playground" | .spec.template.spec.metadata.labels["playground"] += "OK"' |  oc apply -n openshift-machine-api -f -
```

by exporting existing machineset :

```bash
oc get -n openshift-machine-api machineset myopenshiftcluster-worker-eu-central-1c --export -o json  ...
```

then replace .metadata.name by "my-test-playground"
in order to create a new machineset ressource named "my-test-playground"

```bash
... | jq '.metadata.name = "my-test-playground"  ...
```

add labels playground="OK to this machineset ressource

```bash
... | .metadata.labels.playground += "OK" ...
```

remove selflink to avoid collision

```bash
... | del(.metadata.selfLink) ...
```


set .spec.template.metadata.labels["machine.openshift.io/cluster-api-machine-role"] to "my-test-playground"

in order to use a specific cluster-api-machine-role, not mandatory, but it help enforcing segregartion

```bash
... | .spec.template.metadata.labels["machine.openshift.io/cluster-api-machine-role"]="my-test-playground" ...
```

add labels node-role.kubernetes.io/worker="my-test-playground" and playground="OK to the node template. It means that the node create from this machine set will have thoses labels.

```bash
... .spec.template.spec.metadata.labels["node-role.kubernetes.io/worker"] += "my-test-playground" | .spec.template.spec.metadata.labels["playground"] += "OK"'  ...
```

[optional] save the ressource manifest to timed file

```bash
... | tee src/main/openshift/config/tmp-$(date +%y%m%d%H%M%s).json ...

```

apply this ressource manifet to your cluster

```bash
... |  oc apply -n openshift-machine-api -f -

```

This will create a new MachineSet labeled "playground=OK" that will create new node labeled "playground=OK" and "node-role.kubernetes.io/worker=my-test-playground"
It take few minutes to create machine and nodes.
you can watch progress with ```oc get -n openshift-machine-api machineset -l playground=OK -w``` or ```oc get nodes -l playground=OK -w```

### MachineConfig & MachineConfigPools

#### MachineConfig

[MachineConfig](https://docs.openshift.com/container-platform/4.2/architecture/architecture-rhcos.html) are ressources that hold the machine configuration. It use [Ignition](https://github.com/coreos/ignition/tree/spec2x). 

for instance :

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
...
spec:
  config:
    ignition:
      version: 2.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=us-ascii;base64,IyBVc2UgQV...
        filesystem: root
        mode: 420
        path: /etc/...

```

In practice you can obtain the source value with this script :

on OSX 

```bash
echo "data:$(file -bI chrony.conf | tr -d ' ' );base64,$(cat chrony.conf | base64 )"
```

On Linux

```bash
echo "data:$(file -bi chrony.conf | tr -d ' ');base64,$(cat chrony.conf | base64 -w0)"
```

Currently [chrony.conf](src/main/openshift/config/50-worker-testing-conf-chrony.MachineConfig.yaml) and [crio.conf](src/main/openshift/config/50-worker-testing-conf-crio.MachineConfig.yaml) have been done

_Notice_ : MachineConfig are apply sequently so its good practive to use "xx-" namming convention, 99-is reserved to config dynamically created like [KubeletConfigController](https://github.com/openshift/machine-config-operator/blob/master/docs/MachineConfigController.md#kubeletconfig)

#### MachineConfigPools

[MachineConfigPools](https://docs.openshift.com/container-platform/4.2/architecture/architecture-rhcos.html) define the link between Machine and MachineConfig

example :
```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
...
spec:
  configuration:
    name: rendered-worker-playground-
    source:
      - apiVersion: machineconfiguration.openshift.io/v1
        kind: MachineConfig
        name: 50-worker-chrony-playground
    pause: true
  machineConfigSelector:
    matchLabels:
      machineconfiguration.openshift.io/role: playground
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/worker: my-test-playground

```

Configuration is created dynamicaly base on 'machineConfigSelector' and apply dynamicaly based on 'nodeSelector'.

spec.pause allow you to stop the updating process which can be very usefull, specially if you want to confirme configurations

```bash
oc apply -f src/main/openshift/config/playgroung.MachineConfigPool.yaml
```

Look to the detailed MachineConfigPool worker-playground with ```oc describe MachineConfigPool worker-playground```

you'll see :

```yaml
Status:
  Conditions:
...
    Message:               Failed to render configuration for pool worker-playground: no MachineConfigs found matching selector machineconfiguration.openshift.io/role=playground
...
  Configuration:
  Degraded Machine Count:     0
  Machine Count:              1
  Observed Generation:        1
  Ready Machine Count:        0
  Unavailable Machine Count:  0
  Updated Machine Count:      0
```

#### Let's roll

##### let's create the MachineConfig

```bash 
oc apply -f src/main/openshift/config/chrony.MachineConfig.yaml && \
  oc apply -f src/main/openshift/config/crio.MachineConfig.yaml
```

then check that the MachineConfigPool have created a rendered-worker-playground-xxx configuration

```bash
oc describe MachineConfigPool worker-playground
```

```yaml
...
Spec:
  Configuration:
    Name:  rendered-worker-playground-b5a7848873f7aab6bdaf79f4eec21b4a
    Source:
      API Version:  machineconfiguration.openshift.io/v1
      Kind:         MachineConfig
      Name:         50-worker-testing-conf-chrony
      API Version:  machineconfiguration.openshift.io/v1
      Kind:         MachineConfig
      Name:         55-worker-testing-conf-crio
...
   Paused:                              true
...
Status:
  Conditions:

...
  Configuration:
  Degraded Machine Count:     0
  Machine Count:              1
  Observed Generation:        3
  Ready Machine Count:        0
  Unavailable Machine Count:  0
  Updated Machine Count:      0
```

beware that the config must be "complete" this means that in order to have a custum 'machineconfiguration.openshift.io/role=playground' we need to copy the "machineconfiguration.openshift.io/role=worker"

```bash
for MCNAME in $(oc get MachineConfig -l machineconfiguration.openshift.io/role=worker -o name | cut -d '/' -f 2) ; do oc get MachineConfig $MCNAME -o json --export | jq --arg name "$MCNAME-playground" '.metadata.name=$name | del(.metadata.selfLink) | .metadata.labels["machineconfiguration.openshift.io/role"]="playground"' | oc apply -f - ; done
```


##### let's take a fingerprint

Firts of all we'll check ou current configuaration with `oc debug` on target machine. Beware, `oc debug` is like `root ssh`, so use it wisely and remove the pod asap.

```bash
oc debug $(oc get nodes -l node-role.kubernetes.io/master!='',node-role.kubernetes.io/worker=my-test-playground -o name | tail -n 1) &
```

will start a pod in debug mode in backgroung
`oc get nodes -l node-role.kubernetes.io/master!='',node-role.kubernetes.io/worker=my-test-playground -o name` will list node that are not master with label node-role.kubernetes.io/worker=my-test-playground and `tail -n 1` will get the firt element of the list.

let's have a configuration fingerprint before changing it. 

```bash
oc exec -it $(oc get po -o name | cut -d '/' -f 2) -- bash -c "echo "chrony.conf" ; cat /host/etc/chrony.conf | wc -l ; sha256sum /host/etc/chrony.conf ; echo crio.conf; cat /host/etc/crio/crio.conf | wc -l; sha256sum /host/etc/crio/crio.conf"
```

will out put somthing like:

```bash
38
64379900839b05286704b184f42078ee77b4a976ef6bb1ce7a2e62de9409579c  /host/etc/chrony.conf
crio.conf
251
228d19a701aaddaafea6ddd63b7664357ee735bd6622b3b8e800fa14a335552b  /host/etc/crio/crio.conf
```

##### let's start the update

```bash
oc get MachineConfigPool worker-playground -o json | jq '.spec.paused=false' | oc apply -f -
```

<!-- 
```bash 
oc patch MachineConfigPool worker-playground -p '{"spec":{"paused":"false"}}'
Error from server (UnsupportedMediaType): the body of the request was in an unknown format - accepted media types include: application/json-patch+json, application/merge-patch+json
```
-->


```bash
oc get MachineConfigPool worker-playground
NAME                CONFIG   UPDATED   UPDATING   DEGRADED
worker-playground            False     True       False

oc get nodes -l node-role.kubernetes.io/worker=my-test-playground
NAME                                            STATUS                     ROLES    AGE    VERSION
ip-10-0-152-132.eu-central-1.compute.internal   Ready,SchedulingDisabled   worker   132m   v1.14.6+c07e432da
```

and now let's check that our files have been correctly change :

```bash
oc debug $(oc get nodes -l node-role.kubernetes.io/master!='',node-role.kubernetes.io/worker=my-test-playground -o name | tail -n 1) &
oc exec -it $(oc get po -o name | cut -d '/' -f 2) -- bash -c "echo "chrony.conf" ; cat /host/etc/chrony.conf | wc -l ; sha256sum /host/etc/chrony.conf ; echo crio.conf; cat /host/etc/crio/crio.conf | wc -l; sha256sum /host/etc/crio/crio.conf"
chrony.conf
```

that should produce something like that

```bash
7
369eb0b08ffe1b26530a8a73a9b8fcea0c92130cdcd1a5c5e85e1a10c3eef4a9  /host/etc/chrony.conf
crio.conf
65
6f829397e424675c2d856f95770dd75be4c91402d16446bbc625f33d04c45a12  /host/etc/crio/crio.conf
```










