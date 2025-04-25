# Install Disconnected SNO

* Set up mirror registry for disconnected SNO. An example ImageSetConfiguration can be found at imageset-config.yaml
Below Images needs to be mirrored explicitly.
```
  - name: quay.io/openshift_sandboxed_containers/rhcos-layer/ocp-4.16:snp-0.1.0
  - name: quay.io/openshift_sandboxed_containers/kbs:v0.10.1
```
* Develop install-config.yaml using the example given. Update baseDomain, machineNetwork, installationDisk, pullSecret, imageContentSource, sshKey and additionalTrustBundle as needed.

* Develop agent-config.yaml using the example given. Update rendezvousIP, interface name, macAddress and other details appropriately.

* Extract openshift-install from release image on mirror registry, eg

```
oc adm release extract -a pull-secret.txt --icsp-file=<icsp-file.yaml> --command=openshift-install mirror.hub.mylab.com:8443/openshift/release-images:4.18.5-x86_64
```

* Copy the install-config.yaml and agent-config.yaml to a directory eg "ocp" and create the ISO

```
./openshift-install --dir ocp/ agent create image
```
* Boot the SNO node using the ISO to install OCP.

* Apply the manifests to support operators from local mirror registry.

* Disable all default operator sources.
```
oc patch OperatorHub cluster --type json -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
```

# Install and Configure Trustee Operator

* Install OpenShift Trustee.

Install From OpenShift GUI by following the default steps to install an Operator.

Or Install from CLI.
* Create a Namespace, OperatorGorup and Subscription.

```
oc create -f trustee/ns.yaml
oc create -f trustee/og.yaml
oc create -f trustee/subscription.yaml
```
* For disconnected environment, installation of operator may fail which requires editing csv and changing the URL for registry.redhat.io/openshift4/ose-kube-rbac-proxy-rhel9 to use the @sha256 which can be obtained from mirror registry.

* Patch the csv to specify image for Trustee.
```
export TRUSTEE_IMAGE=<mirror-registtry-url:/openshift_sandboxed_containers/kbs:v0.10.1
oc patch -n trustee-operator-system clusterserviceversion.operators.coreos.com/trustee-operator.v0.2.0 --type=json -p="[{"op": "replace","path": "/spec/install/spec/deployments/0/spec/template/spec/containers/1/env/1/value","value": "$TRUSTEE_IMAGE"}]"
```

Alternative
```shell
$ oc edit csv trustee-operator.v0.2.0
# Search for KBS_IMAGE_NAME

```
* Create secrets

```
openssl genpkey -algorithm ed25519 >privateKey
openssl pkey -in privateKey -pubout -out publicKey
```
* Create kbs-auth-public-key secret if it does not exist.
```
oc create secret generic kbs-auth-public-key --from-file=publicKey -n trustee-operator-system
```

* Create KBS configmap
```
oc create -f trustee/kbs-cm.yaml
```

* Create RVPS configmap
```
oc create -f trustee/rvps-cm.yaml
```

* Create resource policy configmap
```
oc create -f trustee/resource-policy-cm.yaml
```

* Create few secrets to serve via Trustee. Create kbsres1 secret only if it doesn't exist. This is just an example. May need to create different secrets for application use cases.
```
oc create secret generic kbsres1 --from-literal key1=res1val1 -n trustee-operator-system
```

* Create KBSConfig
```
oc create -f trustee/kbsconfig.yaml
```
# Install OSC and Configure CoCo

* Install OpenShift SandBox Container Operator. v1.8.1
Install From OpenShift GUI by following the default steps to install an Operator.

Or Install from CLI.
* Create a Namespace, OperatorGorup and Subscription.

```
oc create -f osc/ns.yaml
oc create -f osc/og.yaml
oc create -f osc/subscription.yaml
```
* Create CoCo feature gate ConfigMap
```
oc create -f osc/osc-fg-cm.yaml
```
* Create Layered Image FG ConfigMap. Update the Image location to local mirror for disconnected environment.

```
oc create -f osc/layeredimage-cm-snp.yaml
```

* Create KataConfig
```
oc create -f osc/kata-config.yaml
```

* Wait for the mcp to get fully updated.
```
oc get mcp master
```

* Install NFD Operator.
Install From OpenShift GUI by following the default steps to install an Operator.

Or Install from CLI.
* Create a Namespace, OperatorGroup an Subscription.

```
oc create -f nfd/ns.yaml
oc create -f nfd/og.yaml
oc create -f nfd/subscription.yaml
```
* Edit nfd/nfd-cr.yaml and point the spec.operand.image to point to the image location in the mirror registry. Then apply NFD CR

```
oc create -f nfd/nfd-cr.yaml
```
* Create AMD SNP-SEV rules

```
oc create -f nfd/amd-rules.yaml 
```

* Create Runtime Class
```
oc create -f osc/runtime-class.yaml
```

* Set Agent Kernel Parameters. Export Trustee URL variable. Use the service url if trustee on the same cluster as coco containers. eg http://kbs-service.trustee-operator-system:8080 Use the Route to the svc if trustee is on remote cluster.
```
export trustee_url="Trustee URL"
```

* Set Kernel Paratmer variable
```
export kernel_params="agent.aa_kbc_params=cc_kbc::$trustee_url"
```
* Set the base64 into a variable called source
```
export source=`echo "[hypervisor.qemu]
kernel_params=\"$kernel_params\"" | base64 -w0`
```
* Edit the set-kernel-parameter-kata-agent.yaml and replace the $source with the base64 value from above which can be obtained from "echo $source". Then Create MC for Kernel config
```
oc create -f osc/set-kernel-parameter-kata-agent.yaml
```
