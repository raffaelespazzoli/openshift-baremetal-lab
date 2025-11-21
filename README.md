# openshift-baremetal-lab

these insructions will setup a lab with the following architecture:

![OpenShift baremetal lab architecture](./media/bm-build.drawio.png)

## quick instructions

Start the `Product Enablement: OpenShift Virtualization and Migration Lab` demo lab and log in to the cluster.

Make master schedulable (only needed if using the above lab which is small)

```sh
oc patch schedulers.config.openshift.io cluster --type='json' -p='[{"op": "replace", "path": "/spec/mastersSchedulable", "value":true}]'
```

Enable wildcard routes:

```sh
oc patch ingresscontroller/default -n openshift-ingress-operator --type=merge -p '{"spec":{"routeAdmission":{"wildcardPolicy":"WildcardsAllowed", "namespaceOwnership":"InterNamespaceAllowed"}}}'
```


### deploy kubevirt-redfish

```sh
helm repo add v1k0d3n https://v1k0d3n.github.io/charts
helm repo update
#there is a bug in this chart...
helm install kubevirt-redfish v1k0d3n/kubevirt-redfish -n kubevirt-redfish --create-namespace -f ./kubevirt-redfish/values.yaml
```

### deploy lab helm chart

create a secret with your public key to be able to access the VMs

```sh
oc new-project bm-lab
oc create secret generic mykey --from-file=key1=${HOME}/.ssh/id_rsa.pub -n bm-lab
#this is needed fr the juniper switch image which cannot be freely redistributed
oc create secret docker-registry quay --docker-username=xxx --docker-password=xxx --docker-email=xxx --docker-server=quay.io -n bm-lab
```

```sh
helm upgrade -i bm-lab ./charts/bmh-vm -n bm-lab --create-namespace --set sshPublicKeySecretName=mykey --set password=mypwd
```


connect via ssh to bastion, storage, switch

```sh
kubectl port-forward -n bm-lab service/bastion-ssh 10023:22 &
ssh -p 10023 fedora@localhost

kubectl port-forward -n bm-lab service/storage-ssh 10024:22 &
ssh -p 10024 fedora@localhost

kubectl port-forward -n bm-lab service/switch-ssh 10025:22 &
ssh -p 10025 admin@localhost #pwd admin@123
```

you should also be able to connect with this (currently not fullt tested)

```sh
ssh -o ProxyCommand="openssl s_client -quiet -connect bastion-ssh.sh-proxy-bm-lab.apps.etl6.ocp.rht-labs.com:2222 -servername bastion-ssh.sh-proxy-bm-lab.apps.etl6.ocp.rht-labs.com" fedora@bastion-ssh.sh-proxy-bm-lab.apps.etl6.ocp.rht-labs.com

ssh -o ProxyCommand="openssl s_client -quiet -connect storage-ssh.sh-proxy-bm-lab.apps.etl6.ocp.rht-labs.com:2222 -servername storage-ssh.sh-proxy-bm-lab.apps.etl6.ocp.rht-labs.com" fedora@storage-ssh.sh-proxy-bm-lab.apps.etl6.ocp.rht-labs.com

ssh -o ProxyCommand="openssl s_client -quiet -connect switch-ssh.sh-proxy-bm-lab.apps.etl6.ocp.rht-labs.com:2222 -servername switch-ssh.sh-proxy-bm-lab.apps.etl6.ocp.rht-labs.com" admin@switch-ssh.sh-proxy-bm-lab.apps.etl6.ocp.rht-labs.com #pwd admin@123
```


## uninstallation

```sh
helm uninstall bm-lab -n bm-lab
oc delete namespace bm-lab
```