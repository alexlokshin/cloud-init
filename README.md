# Running Rancher 2.0 + Kubernetes on a standalone ESXi host with vSphere as cloud provider

For us homelab enthusiasts, it might be a real treat to get a decomissioned server off eBay for cheap, load it up with Gigs of 5 year old RAM, and wonder around, marveling the creation. In all actuality, this beast can be turned into a mini cloud, where you can run pretty much anything. Like kubernetes.

Install ESXi 6.7 on your homelab machine. Create a user called `provisioner` with password `somepassword`, grant it the following permissions.

Create a new SSH key with an empty passphrase (https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2), do not overwrite an existing one, ideally save into a separate folder. Make sure you use `RancherOS` for the `-C` parameter value.

Make your cloud-init script accessible on some url (look for an example here: https://raw.githubusercontent.com/alexlokshin/cloud-init/master/rancher.yaml), don't forget to substitute the ssh key with the value from id_rsa.pub. You can also base64 encode the contents of the file  and provide it as a guestinfo parameter called `cloud-init.config.data`.

Create a VM, install docker, install Rancher: https://rancher.com/docs/rancher/v2.x/en/installation/single-node-install/

In Rancher, create a master template, select vSphere as provider, configure the following:

* 1 vCPU core, 8Gb RAM
* vCenter or ESXi Server (provide the IP address of the ESXi host)
* Port (default 443)
* Username: `provisioner`
* Password: `somepassword`

* Add a configuration parameter for guestinfo: `disk.enableUUID` with the value of `TRUE`

Save the template. Create a node template with more vCPUs and RAM.

Proceed to the cluster creation.
* Use vSphere infrastructure provider. Under Cluster options, select Custom cloud provider, then switch to 'Edit as YAML', add the following section to the end:

```yaml
cloud_provider: 
  custom_cloud_provider: "[Global]\nuser = \"provisioner\"\npassword = \"somepassword\"\nport = \"443\"\ninsecure-flag = \"1\"\ndatacenters = \"ha-datacenter\"\nworking-dir = \"kubevols\"\n\n[VirtualCenter \"192.168.88.10\"]\n\n[Workspace]\nserver = \"192.168.88.10\"\ndatacenter = \"ha-datacenter\"\nfolder = \"kubevols\"\ndefault-datastore = \"datastore1\"\n[Disk]\nscsicontrollertype = pvscsi\n[Network]\npublic-network = \"VM Network\""
  name: "vsphere"
```

Please note that `datacenter` and `datacenters` parameters here are mandatory, but for standalone ESXi hosts you can use the default value of `ha-datacenter`.

Click 'Create cluster'.

To test, deploy artifactory from the catalog apps.
