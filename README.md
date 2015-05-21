# MOC Installation Guide

##MOC Setup
  
Follow the guide for the MOC in the link prvided below.

https://github.com/CCI-MOC/moc-public/wiki/EC500-Instructions

Remember the name of the network created. It will be added to the scheduler MOC Config file on the Master Node later.

##Compute Node Setup

###Launching Instance

Launch an Ubuntu Server 14.04 image selecting with a username of ‘chris’ and a machine name and enable ssh password authentication. Be sure the instance has at least 20GB of disk.

###Install Dependencies

**RabbitMQ**

[RabbitMQ](https://www.rabbitmq.com/)  is used by the master and compute node to schedule tasks on the MOC. Below are the installation steps from the website with one small change. Pika version 0.9.9 is used instead of 0.9.8 due to a bug.

>To use RabbitMQ APT repository:
>Add the following line to your /etc/apt/sources.list:
>
   deb http://www.rabbitmq.com/debian/ testing main
>
>To avoid warnings about unsigned packages, add our public key to your trusted key list using apt-key(8):
>
    $ wget https://www.rabbitmq.com/rabbitmq-signing-key-public.asc
    $ sudo apt-key add rabbitmq-signing-key-public.asc
>
>Run apt-get update.
>Install packages as usual; for instance,
>
    $ sudo apt-get install rabbitmq-server
>
>RabbitMQ libraries:
>The installation depends on pip and git-core packages, you may need to install them first.
>
    $ sudo apt-get install python-pip git-core
>
>In this tutorial series we're going to use pika. To install it you can use the pip package management tool:
>
    $ sudo pip install pika==0.9.9

**MOC scheduling:**
	
Clone scheduler scripts from github to the directory /home/chris/src/rabbitmq. 

**chrisreloaded:**

* Run compute-install-packages.sh file from the cloned rabbitmq directory to install several required packages.

* Use git to clone the FNNDSC ‘chrisreloaded’ and ‘scripts’ github projects into /home/chris/src and clone _common into the /home/chris/src/chrisreloaded/lib directory.

* Create directory called users in /home/chris/ and create symbolic link in /home/chris/src/chrisreloaded also called users.

* Add the freesurfer files to /home/chris/arch/Linux64/packages directory and create a symbolic link to the Linux64 directory 
in /home/chris/arch called x86_64-Linux.

* Create directory /neuro/tmp/ and make the permissions 777 on both.

**SSH Keys**

Generate and append a ssh key to the compute nodes own .ssh/authorized_keys file. Ensure the compute node can passwordless
ssh into itself.

**Enable Worker Script to Run at Startup**

To listen for tasks automatically on start up add to /etc/rc.local file:

	    su - chris -c /home/chris/src/rabbitmq/chris_worker.py &

##Master Node Setup

###Launch Instance
Launch an Ubuntu Server 14.04 image with a username ‘chris’ and a machine name and enable ssh password authentication.
Assign a floating IP to the instance.

###Install Dependencies
**Install python numpy and scipy:**

    $ sudo apt-get install python-numpy python-scipy

**Setup RabbitMQ and Pika:**

Follow the same steps as the Compute Node installation of RabbitMQ. After the rabbitmq and pika dependencies are installed,
enter the following commands in the command line  to create a user 'chris' with the password 'chris1234' and a virtual
host 'master' on the master node.
 
    # add new user
    $ sudo rabbitmqctl add_user chris chris1234

    # add new virtual host
    $ sudo rabbitmqctl add_vhost master

    # set permissions for user on vhost
    $ sudo rabbitmqctl set_permissions -p master chris ".*" ".*" ".*"

    # restart rabbit
    $ sudo rabbitmqctl restart

**Scheduling Scripts:**

Clone the MOC rabbitmq scripts from github to the directory /home/chris/src/rabbitmq. In the chris-moc.cfg file input 
the network name from the MOC Setup above and the volume snapshot name created in the Compute Node Setup to use for 
booting up new compute nodes.

**Setup Python-novaclient:**

Python-novaclient is the package used to launch and terminate compute instances on the MOC. To install run the following
command.

    $ sudo pip install python-novaclient

An Openstack openrc file is needed for the essential environment parameters to run the novaclient commands on the Openstack 
API. Download it from the Openstack “Compute->Instances->Access & Security->API Access->Download Openstack RC File”.
Add this file to the master node /home/chris/src/rabbitmq directory, rename it to “chris-openrc.sh”, and make it executable. 
Open the chris-openrc.sh script and change the following lines in the script with  “********” being your OpenStack login
password. 

    #echo "Please enter your OpenStack Password: "
    #read -sr OS_PASSWORD_INPUT
    #export OS_PASSWORD=$OS_PASSWORD_INPUT
    export OS_PASSWORD="********"

**Clone chrisreloaded and _common files:**

For the Master Node to take requests from the ChRIS dashboard clone chrisreloaded files from github to the directory
/home/chris/src and clone _common files from github to directory /home/chris/src/chrisreloaded/lib.

###Enable Scaler Script to Run at Startup

To provide automatic scaling the scaler script is run automatically on start up by adding to /etc/rc.local file:

  su - chris -c /home/chris/src/rabbitmq/chris_scaler.py &
  
Wait unitl the MOC Config file has been configured before rebooting the Master Node to start the scaling.
  
###SSH Setup

Passwordless ssh is required between the Compute Nodes and the Master Node in both directions. Append the ssh public key from the Compute Node to /home/chris/.ssh/authorized_keys on the Master Node. Do the reverse as well appending the ssh public key from the Master Node to the ~/.ssh/authorized_keys on the Compute Node.

Passwordless ssh is required between the ChRIS webhost and the Master Node in both directions. Append the ssh public key from the ChRIS webhost to /home/chris/.ssh/authorized_keys on the Master Node. Do the reverse as well appending the ssh public key from the Master Node to the ~/.ssh/authorized_keys on the ChRIS webhost.

To allow reverse tunneling from the compute nodes open the file ‘/etc/ssh/sshd_config’ and add to the last line:

    GatewayPorts yes

##MOC Config File

This config file on the Master Node contains the parameters necessary for MOC scaling and scheduling. Once all the parameters 
below have been added reboot the Master Node for the scaling to take in to effect.

###MOC Network Name

On the Master Node, input the network name from the MOC Setup to /home/chris/src/rabbitmq/chris-moc.cfg.

###Volume Snapshot

For fast scaling of nodes a volume snapshot is needed of the Compute Node. In the Openstack Dashboard snapshot the Compute Node instance. This will take 30 min to an hour. When the snapshot is finished create a volume from the snapshot, then create a volume snapshot from the volume. Once the volume snapshot has completed, click on it for an overview. In the overview find the the volume snapshot id and add this to the volume_snapshot_id on the Master Node /home/chris/src/rabbitmq/chris-moc.cfg file.

###Scaling Parameters

Automatic scaling is handled by the chris_scaler.py script on the Master Node and has several parameters associated with it.
This script monitors the length of the task queue assigned by the ChRIS dashboard and the number of Compute Nodes currently
running. Scaling parameters can be set in /home/chris/src/rabbitmq/chris-moc.cfg. 
* The variable **min_workers** establishes the minimum number of compute workers on the MOC at any time. 
* The **ceil_threshold** parameter determines at what ratio of assigned tasks to workers should the number of Compute Nodes scale up. 
* The **scale_by** parameter determines the percentage by which to scale up by based on the current number of Compute Nodes. 
* The **floor_threshold** parameter determines at what ratio of assigned tasks to workers should the number Compute Nodes scale down.

##ChRIS Web Host Setup
###Cluster Host

SSH tunneling is used so CLUSTER_HOST will be the name of the ChRIS web host and the CLUSTER_PORT will be the tunneling port. Be sure that the CLUSTER_HOST name is in the /etc/hosts file under 127.0.0.1.
