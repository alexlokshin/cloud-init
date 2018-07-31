# Running Rancher 2.0 + Kubernetes on a standalone ESXi host with vSphere as cloud provider

For us homelab enthusiasts, it might be a real treat to get a decomissioned server off eBay for cheap, load it up with Gigs of 5 year old RAM, and wonder around, marveling the creation. In all actuality, this beast can be turned into a mini cloud, where you can run pretty much anything. Like kubernetes. This text below depicts the steps you need to take in order to take advantage of volume provisioning along with VM provisioning using Rancher 2.0. It's a bit different from the official documentation, and combines bits and pieces from all over, along with my own discoveries.

Install ESXi 6.7 on your homelab server (I recommend at least 8 phusical CPU cores, and at least 96Gb of RAM). After ESXi is installed, navigate to the ESX_IP provided on the yellow and grey screen. Log in with `admin` and the password you've selected during setup, join all disks into a datastore called `datastore1` (that is a default name, anyways), and create a user called `provisioner` with password `somepassword`, grant it the following permissions (that can be done by right clicking `Host`, and selecting `Permissions`.

To be able to use the same SSH key to connect to all of the kubernetes nodes, create a new SSH key with an empty passphrase (https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2), do not overwrite an existing one, ideally save into a separate folder. Make sure you use `RancherOS` for the `-C` parameter value.

Make your cloud-init script accessible on some url (look for an example here: https://raw.githubusercontent.com/alexlokshin/cloud-init/master/rancher.yaml), don't forget to substitute the ssh key with the value from id_rsa.pub. You can also base64 encode the contents of the file  and provide it as a guestinfo parameter called `cloud-init.config.data`.

Create a VM (I recommend using a minimal CentOS 7.x images as a base: http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1804.iso), install docker (https://docs.docker.com/install/linux/docker-ce/centos/), install Rancher: https://rancher.com/docs/rancher/v2.x/en/installation/single-node-install/.

To begin the cluster creation process, you will need to select two node templates, one for the control plane (master nodes), one for the worker nodes. The difference between the two templates is in the amount of CPU and RAM you're willing to allocate. Control plane nodes need a lot less of those. You can run all nodes as a mix of control plane and worker, but make sure you have an odd number of them (same applies to pure control plane).

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
  custom_cloud_provider: "[Global]\nuser = \"provisioner\"\npassword = \"somepassword\"\nport = \"443\"\ninsecure-flag = \"1\"\ndatacenters = \"ha-datacenter\"\nworking-dir = \"kubevols\"\n\n[VirtualCenter \"ESX_IP\"]\n\n[Workspace]\nserver = \"ESX_IP\"\ndatacenter = \"ha-datacenter\"\nfolder = \"kubevols\"\ndefault-datastore = \"datastore1\"\n[Disk]\nscsicontrollertype = pvscsi\n[Network]\npublic-network = \"VM Network\""
  name: "vsphere"
```

Here, `ESX_IP` is the IP address of your standalone ESXi host.

Please note that `datacenter` and `datacenters` parameters here are mandatory, but for standalone ESXi hosts you can use the default value of `ha-datacenter`.

Click 'Create cluster'.

To test, deploy artifactory from the catalog apps.

If it doesn't work, look through the kubelet logs. That's what the common ssh key is for. SSH into the master node, run `docker ps | grep kubelet`, get the container id. Run `docker logs <CONTAINER_ID>`, and look for errors.

Recap. What did we end up with? Kubernetes can provision dynamic volumes, and Rancher can create and replace failed nodes. You're constrained by the amount of disk, CPU and RAM available. At the moment, the best value can be had by procuring machines like Dell PowerEdge r610/620, Dell PowerEdge R710/720, as well as HP ProLiant Gen7/8. 

PS. Same approach can be used for a real vSphere in an enterprise setting, changes are rather minimal.
