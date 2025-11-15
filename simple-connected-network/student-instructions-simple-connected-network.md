# Student instructions for cluster installation

This instructions ar for the simpe and connected network

Assuming you are logged in into the cluster and pointing to your namespace, run the folowing instructions

## Preparation

execute this from your laptop and from the git repo folder

```sh
export ssh_key=$(cat ~/.ssh/id_rsa.pub) #replace here with the key you want to use.
export pull_secret=$(oc get secret pull-secret -n openshift-config -o go-template='{{index .data ".dockerconfigjson" | base64decode}}')
for node in master1 master2 master3 worker1 worker2 worker3; do
    export "${node}_macaddress"=$(oc get vm ${node} -o jsonpath='{.spec.template.spec.domain.devices.interfaces[0].macAddress}')
done
mkdir install-files
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
    system_id=$(curl -k -G "http://${server}-bmc:8000/redfish/v1/Systems/" | jq -r '.Members.[0]."@odata.id"')
    curl -v -k http://${server}-bmc:8000${system_id}/VirtualMedia/Cd/Actions/VirtualMedia.InsertMedia -H 'Content-Type: application/json' -d '{"Image": "http://bastion-ssh:80/agent.x86_64.iso", "Inserted": true}'
done

for server in master1 master2 master3 worker1 worker2 worker3; do 
    system_id=$(curl -k -G "http://${server}-bmc:8000/redfish/v1/Systems/" | jq -r '.Members.[0]."@odata.id"')
    curl -d '{"ResetType":"On"}' -H "Content-Type: application/json" -X POST http://${server}-bmc:8000${system_id}/Actions/ComputerSystem.Reset
done
```

```sh
for server in master1 master2 master3 worker1 worker2 worker3; do
    session_id=$(curl -i -X POST -H "Content-Type: application/json" http://bm-lab-${server}-virtbmc.kubevirtbmc-system.svc.cluster.local/redfish/v1/SessionService/Sessions -d '{"UserName":"admin","Password":"password"}' | | grep X-Auth-Token | awk '{print $2}')
    system_id=$(curl -k -G "http://${server}-bmc:8000/redfish/v1/Systems/" | jq -r '.Members.[0]."@odata.id"')
    curl -H "X-Auth-Token: 1f555f911c02f8fa93290c4201d9f6039990089aa0b89ca66bf8dbf80ef01928" http://bm-lab-${server}-virtbmc.kubevirtbmc-system.svc:/redfish/v1/Systems/1/VirtualMedia/Cd/Actions/VirtualMedia.InsertMedia -H 'Content-Type: application/json' -d '{"Image": "http://bastion-ssh:80/agent.x86_64.iso", "Inserted": true}'
done

for server in master1 master2 master3 worker1 worker2 worker3; do 
    system_id=$(curl -k -G "http://${server}-bmc:8000/redfish/v1/Systems/" | jq -r '.Members.[0]."@odata.id"')
    curl -d '{"ResetType":"On"}' -H "Content-Type: application/json" -X POST http://${server}-bmc:8000${system_id}/Actions/ComputerSystem.Reset
done
```

### follow installation logs

```sh
./openshift-install --dir ./bm-lab-installation agent wait-for bootstrap-complete --log-level=info
openshift-install --dir ./bm-lab-installation agent wait-for install-complete
```