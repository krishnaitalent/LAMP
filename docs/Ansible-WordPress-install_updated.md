# Installing WordPress with WooCommerce on a LAMP Cluster in Microsoft Azure
This document explains how to install WordPress with the WooCommerce plugin on a LAMP cluster in Azure. 
The following diagram shows how the required system components participate in the installation process.

![Workflow](https://github.com/krishnaitalent/LAMP/blob/lamp_docmentation/images/WordPress_Flow_Diagram.png)

## Prerequisites

To install WordPress with the WooCommerce plugin, you must prepare your system environment as follows:
- Deploy a LAMP stack (the Controller VM will be created that will host the WordPress with WooCommerce installation). To support the current installation, the LAMP stack must be running:
	*	Ubuntu 16.04 LTS
	*	Nginx web server 1.10.3
	*	MySQL PaaS 5.6, 5.7 or 8.0 database server
	*	PHP 7.2, 7.3, or 7.4 
- Make sure the Ansible VM is located in the same Azure resource group and region that hosts the Controller VM.

## Overview of the Installation Process
- The installation process consists of the following high-level procedures:
	*	Enable Password Authentication on the Controller VM
	*	Deploy the Ansible VM
	*	Install WordPress with WooCommerce on the Controller VM


### Enable Password Authentication on the Controller VM
Enabling password authentication allows you to obtain the Controller VM credentials and copying SSH keys that you will need to deploy the Ansible VM later in the installation process.

- Login to the Controller VM and navigate to the /home/azureadmin(user)/ directory, where azureadmin(user) represents the username of the Azure admin account.
- Run the following commands to enable password authentication.

```
sudo sed -i "s~PasswordAuthentication no~PasswordAuthentication yes~" /etc/ssh/sshd_config
sudo sed -i "s~#UseLogin no~UseLogin yes~" /etc/ssh/sshd_config
sudo sed -i "s~#   StrictHostKeyChecking ask~   StrictHostKeyChecking no~" /etc/ssh/ssh_config
sudo systemctl restart sshd
sudo passwd azureadmin
```
- When prompted, configure a new password for the Controller VM.
- Gather the following information from the Controller VM:
	* Controller VM username
	* Controller VM IP address
	* Controller VM password (the new password that you configured in the previous step)

### Deploy the Ansible VM to Install WordPress with WooCommerce:

The Ansible VM will be used to execute the WordPress installation script. To deploy the Ansible VM, you use the ARM Template and provide input parameters describing the Controller VM and the new database that will be created during the installation of WordPress.

- Make sure that you are deploying Ansible VM into the same Azure resource group and region as the Controller VM.
- Use the [ARM Template](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fummadisudhakar%2FLAMP%2Fansible_playbook_mat32%2Fansibledeploy-wordpress.json) to start the Ansible VM deployment.
- Enter the Controller VM information that you gathered earlier:
	* Ansible VM name
	* SSH public key
	* Controller VM username
	* Controller VM IP address
	* Controller VM password
- Enter the information for the new WordPress database:
	*	Server name of the MySQL server in the LAMP cluster
	*	Login username for the MySQL server
	*	Login password for the MySQL server
	*	Name of the new WordPress database
- Enter additional configuration information:
	*	Domain name of the WordPress instance
	*	Domain of the load balancer.

- The following processes occur during the deployment:
	*	The ARM Template executes the wordpress_main.sh script from the extension’s. 
	*	The wordpress_main.sh script downloads the Ansible script (wordpress_script.sh) from GitHub.
	*	The downloaded wordpress_script.sh script is placed in the /home/azureadmin(user) directory on the Ansible VM, where azureadmin(user) represents the username of the Azure admin account.
###
The Ansible VM extension executes the wordpress_script.sh script, which installs WordPress and the WooCommerce plugin using an Ansible playbook.

- The run.sh file runs the Ansible script wordpress_script.sh consisting of an Ansible playbook with the following roles:

	*	Role: SSH Key Configuration 
		* Generates the SSH key pair and copies the key pair to the Controller VM using password authentication.
	*	Role: WordPress 
		*	Downloads and installs WordPress on the Controller VM.
	*	Role: WooCommerce 
		*	Downloads and installs the WooCommerce plugin on the Controller VM.
	*	Role: Replication 
		*	This role executes the following functions.
			*	Configure SSL Certs: Generates the required OpenSSL certificates and places them in the /azlamp/certs/ directory.
			*	Linking Data Location: Links the data directory (/azlamp/data/) to all shared web frontend instances.
			*	Create Nginx Configuration: Creates the Nginx configuration file in the Virtual Machine Scale Set (VMSS) instance.
			* Replication: Replicates the WordPress directory to the VMSS instance for high availability.

When all the roles are executed, the installation is complete.

### Accessing WordPress
To open WordPress, connect to the Azure load balancer using its IP address. You can obtain the load balancer IP address from the Azure Portal.

To log in to WordPress, use the username and password provided in the wordpress.txt file at /home/azureadmin(user), where azureadmin(user) represents the username of the Azure admin account.