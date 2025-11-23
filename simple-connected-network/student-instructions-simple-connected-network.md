# Student instructions for cluster installation

This instructions ar for the simpe and connected network

Assuming you are logged in into the cluster and pointing to your namespace, run the folowing instructions

## Preparation

execute this from your laptop and from the git repo folder

```sh
mkdir install-files
ssh-keygen -t rsa -f ./install-files/id_rsa
export ssh_key=$(cat ./install-files/id_rsa.pub)
export pull_secret=$(oc get secret pull-secret -n openshift-config -o go-template='{{index .data ".dockerconfigjson" | base64decode}}')
for node in master1 master2 master3 worker1 worker2 worker3; do
    export "${node}_macaddress"=$(oc get vm ${node} -o jsonpath='{.spec.template.spec.domain.devices.interfaces[0].macAddress}')
done
envsubst < ./charts/install-manifests/values.yaml > ./install-files/values.yaml
helm template bm-lab ./charts/install-manifests -f ./install-files/values.yaml --output-dir ./install-files
echo $pull_secret > ./install-files/install-manifests/templates/pullsecret.json
```    

## on the bastion

now we ssh to the bastion, execute the following commands

### iso prepareation

```sh
kubectl port-forward -n bm-lab service/bastion-ssh 10023:22 &
scp -P 10023 ./install-files/install-manifests/templates/* fedora@localhost:/home/fedora
scp -P 10023 ./install-files/id_rsa fedora@localhost:/home/fedora/.ssh/ 
ssh -p 10023 fedora@localhost
oc adm release extract --command openshift-install quay.io/openshift-release-dev/ocp-release:4.20.3-x86_64 --registry-config ./pullsecret.json
sudo mv openshift-install /usr/local/bin/openshift-install
mkdir -p ~/bm-lab-installation
cp agent-config.yaml install-config.yaml ./bm-lab-installation
openshift-install --dir ./bm-lab-installation agent create image
sudo mv bm-lab-installation/agent.x86_64.iso /var/www/html/
sudo chgrp -R apache /var/www/html
restorecon -Rv /var/www/html
# ensure the iso is ready to be served:
curl -I http://localhost/agent.x86_64.iso
```

### ISO mount and servers reset

```sh
for server in master1 master2 master3 worker1 worker2 worker3; do
    curl -u admin:changeme -v -k http://kubevirt-redfish.kubevirt-redfish.svc.cluster.local:8443/redfish/v1/Systems/${server}/VirtualMedia/Cd/Actions/VirtualMedia.InsertMedia -H 'Content-Type: application/json' -d '{"Image": "http://bastion-ssh:80/agent.x86_64.iso", "Inserted": true}'
done
```

### follow installation logs

```sh
openshift-install --dir ./bm-lab-installation agent wait-for bootstrap-complete --log-level=info
openshift-install --dir ./bm-lab-installation agent wait-for install-complete
```

### connect to a node to troubleshoot

```sh
ssh core@[node] 
```