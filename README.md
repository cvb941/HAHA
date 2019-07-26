# HAHA - Highly Available Home Assistant
A [Docker swarm](https://docs.docker.com/engine/swarm/) solution allowing to run Home Assistant in a highly available, failover configuration. 

Uses [Ansible](https://www.ansible.com/) playbooks for setup and removal of the cluster on target devices. [GlusterFS](https://www.gluster.org/) for file synchronization between the nodes.

Deploys a fully redundant Home Assistant stack with configured [MariaDB Galera Cluster](https://galeracluster.com/) and [Mosquitto](https://mosquitto.org/) MQTT broker.

## How it works
This project provides a docker-compose file which deploys a stack with Home Assistant, MariaDB, Mosquitto and optionally Portainer containers.

Even though by itself, Docker provides a failover capability by rescheduling failed containers, it doesn't transfer any state between them. This is important, as without state transfer the new containers would simply start from scratch, everytime a new container would be started.

To achieve state transferring for a stateful application such as this, this project configures a realtime file-based synchronization between the nodes for the Home Assistant directory and Mosquitto retain files, using the network filesystem [GlusterFS](https://www.gluster.org/).

As for the MariaDB database used for the recording of history in Home Assistant, the [Galera Cluster](https://galeracluster.com/) is setup and used in a master/master configuration, synchronizing the databases throughout the cluster.

### Connecting sensors and devices to the cluster using MQTT
This cluster includes the [Mosquitto](https://mosquitto.org/) broker for communicating with devices through MQTT. Even though the broker is accessible from any node in the Docker swarm through the swarm's [overlay network](https://docs.docker.com/network/overlay/), a sensor still needs to connect to one of the nodes in the first place.

In the sensor's configuration, we could specify the MQTT broker's address to be just one of the nodes IP addresses. But in case that specific node fails, we could no longer connect to the swarm at all, and thus could not connect to the broker.

A solution would be to include all of the nodes' addresses in the sensor's firmware, and have some kind of a round-robin trial and error in trying to connnect to one of them. However, this list of addresses could be difficult to manage with large amounts of sensors, and some systems, like [ESPHome](https://esphome.io/index.html), do not support multiple MQTT broker addresses.

Other than setting up an own DNS server, a working solution is to set the **same hostnames** to all of the nodes. This way, a sensor will always resolve an IP address, through mDNS, to one of the nodes.

Experimentally, when turning of the node to which a sensor, flashed with [ESPHome](https://esphome.io/index.html), was connected, the sensor was stuck in trying to resolve a different IP address. A solution to this is to restart the Avahi service on any of the running nodes and the sensor magically connects. Because of this, a cron job is setup with [Ansible](https://www.ansible.com/) to restart the Avahi service every minute.


## Limitations
To have a fully autonomously recoverable setup, a **minimum of three devices** need to be part of the cluster. This is due to how both [Docker swarm](https://docs.docker.com/engine/swarm/) and [GlusterFS](https://www.gluster.org/) work. Simply put, all of the decisions in both systems need to be made with the majority of the nodes in consensus. If there were only two nodes, just one of them failing would already cause the majority to be lost. More info on the algorithm can be found [here](https://docs.docker.com/engine/swarm/raft/ "Raft consensus in swarm mode") and [here](https://docs.gluster.org/en/latest/Administrator%20Guide/arbiter-volumes-and-quorum/#client-quorum "Gluster Docs - Client Quorum").

At this point, this project is made to be used, and has only been tested with Raspberry Pi. Support for other devices and architecture could be provided hopefully easily enough, and is just a matter of finding the right Docker images and testing.

## Running the cluster
Although effort has been put to make the setup easy to execute by using the [Ansible](https://www.ansible.com/) tool, there are some manual steps that need to be made.

### Cluster options
You can change the [MariaDB](https://mariadb.org/) passwords in the files located in the *.secrets* directory, all passwords are `pass123` by default.

Change the username of the [MariaDB](https://mariadb.org/) database in the `docker-compose.yml` file. Default username is `user123`. Be careful, there are two services in the file, `mariadb-seed` and `mariadb-node`.

### Preparing the nodes
Make sure the nodes have the Docker Engine installed and ready. Also setup SSH if you plan on connecting to them from [Ansible](https://www.ansible.com/) through it.

### Ansible inventory setup
Create `hacluster` group in the `/etc/ansible/hosts` file. Add your cluster nodes to it. Instructions on how to do so can be found [here](https://docs.ansible.com/ansible/latest/network/getting_started/first_inventory.html).

```
# Example hacluster group definition. Append this to your /etc/ansible/hosts inventory file.

[hacluster]
192.168.1.100  ansible_connection=ssh  ansible_ssh_user=pi  ansible_ssh_pass=raspberry
192.168.1.101  ansible_connection=ssh  ansible_ssh_user=pi  ansible_ssh_pass=raspberry
192.168.1.102  ansible_connection=ssh  ansible_ssh_user=pi  ansible_ssh_pass=raspberry
```

### Setup HAHA cluster playbook
This playbook adds the nodes to a trusted Gluster pool, creates a Gluster volume and mounts it on all nodes. It then copies the default configuration files for Home Assistant and Mosquitto. Then, it sets up a cron job on all nodes to periodically restart the Avahi service. Finally, it initializes a Docker swarm, joins all nodes to it and starts the HAHA stack on it. Run the playbook using the command bellow.

`$ ansible-playbook setup-hacluster.yml`

### Initializing the Galera Cluster
After running the [Ansible](https://www.ansible.com/) playbook, the cluster is not ready for use yet. The Galera Cluster requires some manual initialization. Follow the steps below. For these, you can use the provided [Portainer](https://www.portainer.io/) container manager, included in the `docker-compose.yml`.

1. Wait until the initialization of the container `mariadb-seed` succeeds (container is healthy)
2. Raise the number of replicas of the `mariadb-node` service from zero to the number of nodes in the cluster.
3. After all of the `mariadb-node` containers are done initializing (marked as healthy), remove the `mariadb-seed` service (set replicas to zero).
4. Set the number of replicas of the `homeassistant` service to one.

After this, the Home Assistant container should initialize and be ready to use on the port 8123 in the swarm.

### Remove HAHA cluster playbook
This playbook stops the HAHA stack, unmounts the gluster volumes and removes them. It also cancels the cron job for restarting the Avahi service. Run the following command:

`$ ansible-playbook remove-hacluster.yml`

## Extremely high availability mode
When reacting to a failure inside the cluster, the Docker swarm scheduler might take a while to start and initialize a new Home Assistant or Mosquitto container. More so, if the new node doesn't have the Docker image it needs, in which case it needs to download it.

A solution is to speculatively run multiple instances of Home Assistant at the same time, so that the backup is always running. For that, you need to raise the replicas of the `homeassistant` service to two. In background, both instances will write to the same storage backend in the [GlusterFS](https://www.gluster.org/) volume, which might seem troubling at first, but in practice there weren't any apparent problems.

Running two instances of the [Mosquitto](https://mosquitto.org/) broker simultaneously would require bridging them, which is not a part of this project. An alternative could be to use the ESPHome's [Native API](https://esphome.io/components/api.html) as a replacement of MQTT.

## Running with 2 devices
Although unsupported by both [Docker swarm](https://docs.docker.com/engine/swarm/) and [GlusterFS](https://www.gluster.org/) due to reasons explained in the [limitations](#limitations) section, the cluster *can* technically run on just two nodes.

However, you will lose the automatic recovery of using 3+ devices. Docker swarm will fail to start replacement containers in case of a node failure, so you will need to apply the principles from the [extremely high availabilty](#extremely-high-availability-mode) section to make sure that the system will stay functioning.

Moreover, a split brain scenario may occur in the [GlusterFS](https://www.gluster.org/) volume if only one of two nodes are online. The volume will still be working, but a [manual recovery](https://docs.gluster.org/en/latest/Troubleshooting/resolving-splitbrain/) will be needed when re-adding the lost node.


## Credits
[Colin Mollenhour](https://github.com/colinmollenhour) - For his [amazing solution](https://github.com/colinmollenhour/mariadb-galera-swarm) for managing a MariaDB Galera Cluster in auto-scheduling systems, such as Docker swarm.

User [quasar66](https://community.home-assistant.io/u/quasar66) - For [describing his own HA setup](https://community.home-assistant.io/t/is-it-possible-to-configure-several-homeassistant-docker-containers-to-act-as-a-docker-swarm/38359/37) which served as an inspiration for this project.