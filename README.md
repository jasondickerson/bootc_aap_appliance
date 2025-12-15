# BootC AAP Appliance

## Description

This repository is an example of how to Create an AAP 2.5 containerized Edge Appliance.  The OS is RHEL 9 image mode and the OS images are maintained as Containers.  The current OS version is RHEL 9.6, but this also works for RHEL 10.0.  The current AAP version is Containerized 2.5-18.  The included playbooks deploy a Growth Topology or All in One AAP; however this configuration can be used to deploy other topologies by manually configuring an inventory and configuring SSH keys.  

## Build your own OS image and provisioning ISO

### Prerequisites

* RHEL 9 or 10 VM
* Install ansible-core and git
  
  ```shell
  # sudo dnf install -y ansible-core git
  ```

* Clone this repository down to the VM

  ```shell
  # git clone ssh://git@gitlab.consulting.redhat.com:2222/jdickers/bootc_aap_appliance.git
  ```

* Install the required Ansible collections

  ```shell
  # cd bootc_aap_appliance
  # ansible-galaxy collection install -r requirements.yml
  ```

### Configure playbook variables

#### vars/main.yml

  ```text
  redhat_offline_token:    Red Hat API Offline Token
                           https://access.redhat.com/management/api
                           Used to download the RHEL Boot ISO and the Containerized AAP Setup Bundle

  registry_username:       Red Hat Registry Service Account or Red Hat Online Username
                           https://access.redhat.com/terms-based-registry/accounts
                           Used to download the BootC base OS container image.

  registry_password:       Red Hat Registry Service Account or Red Hat Online Password
                           https://access.redhat.com/terms-based-registry/accounts
                           Used to download the BootC base OS container image.

  aap_os_image_name:       Name of the OS container image to create.
  
  aap_os_image_version:    Version tag of the OS container image to create.
  
  rhel_iso_version:        RHEL version of the Boot ISO to download.  
  
  os_disk:                 Device name of the disk to install the OS to.
                           libvirt VirtIO: vda
                           scsi: sda
  
  password_hashes.ansible: Password hash for the OS user AAP will run as.
  
  password_hashes.root:    Password hash of the OS root user.
  ```

Example:

  ```yaml
  ---
  # From https://access.redhat.com/management/api
  redhat_offline_token: ''
  # From: https://access.redhat.com/terms-based-registry/accounts
  registry_username: ''
  registry_password: ''
  aap_os_image_name: aap_edge
  aap_os_image_version: "1.0"
  rhel_iso_version: "9.6"
  os_disk: vda
  password_hashes:
    ansible: '$6$rounds=100000$ZywuBVmjYEYUSEAr$BvikY3M3DwgUD3DYwPJkxg2BpMrirp7QhO1SbLVzwtS4rNgM2YwOjtrDinbn/DQV8NwMtWg7D.o/PgITincpE1'
    root: '$6$rounds=100000$ZywuBVmjYEYUSEAr$BvikY3M3DwgUD3DYwPJkxg2BpMrirp7QhO1SbLVzwtS4rNgM2YwOjtrDinbn/DQV8NwMtWg7D.o/PgITincpE1'
  ...
  ```

> NOTE: When booting the ISO, as part of the OS deployment process, the first disk is wiped and the OS is installed to it.  If the device name in the os_disk variable (vars/main.yml line 10) does not exist on the host being deployed to, the process will hang.  By using this variable, any disk device type can be targeted.  The variable should only point to the first disk on the system, and the RHEL OS image will only consume the that disk.  If you have additional disks available, it is possible to manually allocate the additional disks post deployment.

### Configure OS Configuration and AAP deployment default variables

#### ansible_playbooks/vars/default.yml

  ```text
  aap_user:                  OS user AAP will run as.

  aap_version:               Version of AAP to deploy

  aap_installer_dir:         Directory containing the expanded Containerized AAP Setup Bundle
                             Not recommended to change.

  ansible_installer_timeout: Timeout to wait for AAP installer actions to complete
                             Not recommended to change, unless deploying on very slow hardware.

  aap_fqdn:                  FQDN of AAP
                             Do not change, this is picked up by the OS deployment process from another file.

  aap_hostname:              AAP FQDN and port used to authenticate to AAP for configuration post install.
                             Do not change.

  aap_username:              AAP admin user to authenticate to AAP for configuration post install.
                             Do not change.

  aap_validate_certs:        Disable certificate validation when configuring AAP post install.
                             Do not change as AAP deploys with self signed certificates.
  ```

Example:

  ```yaml
  ---
  aap_user: ansible
  aap_version: '2.5-18'
  aap_installer_dir: "/var/opt/ansible-automation-platform-containerized-setup-bundle-{{ aap_version }}-x86_64/"
  ansible_installer_timeout: 3600
  aap_fqdn: "{{ hostname }}.{{ domain }}"
  aap_hostname: "{{ aap_fqdn }}:443"
  aap_username: admin
  aap_validate_certs: false
  ...
  ```

> NOTE: When using the Red Hat API to download the Containerized AAP Setup Bundle, only the latest version of the Containerized Setup Bundle is available.  If you are not specifying the latest Containerized Setup Bundle, the playbook will fail with an error indicating the specified Containerized Setup Bundle is not available, and then list the version that is available.  If you wish to use the currently specified Containerized Setup Bundle, you will need to skip the download and have the desired version available.  If you wish to use the latest Containerized Setup Bundle, update ansible_playbooks/vars/default.yml line 3, aap_version, with the version listed as available from the playbook failure.  

### Create the OS image and Installation ISO

To create the OS image in the local podman repository and the Provisioning ISO run the following command:

```shell
# cd bootc_aap_appliance
# ansible-playbook build_iso.yml
```

If desired, it is possible to add a 'podman push' task to the playbook in order to upload the newly build BootC OS container image to a container registry of your choosing.  

### Deployment

To deploy the AAP appliance you will need a Physical or Virtual server with:

* x86_64 architecture
* 4 CPU
* 16GB RAM
* 256GB HDD recommended, 128GB Minimum

Boot the system using the ISO file.  The ISO will:

* Wipe the first disk (specified in the os_disk variable)
* Install the OS image
* Reboot

Once the system is online you can login with one of the two OS users:

* maint   - An unprivileged maintenance user with access to run certain commands with elevated privileges.  
* ansible - A privileged Administrative user with sudo access.  This is the AAP user.

The root user should not be used unless absolutely necessary, and can only be used from the local console, not from ssh.  

The passwords for the root and ansible os users are set in vars/main.yml as password hashes.  The hashes may be replaced with another desired password hash.  The included password is 'changeme' for both users.  

The default password for maint is 'changeme', which is set in the templates/local.ks.j2.  The intent is that the password would be changed to a unique password upon deployment.  

The ansible user is the user AAP will run as and is a full System Administrator via sudo.  

### Configure Networking and Install AAP

The example below shows how to use the maint user to perform the initial OS configuration and AAP installation.  This user is configured for a "true appliance" use case.  Where the appliance is provided to a customer that will not have admin privileges, but rather be able to use the AAP for pre-configured automation.  

As an alternative you may choose to login as the ansible user and run the playbook as the privileged user.  It is located in /var/home/ansible/ansible_playbooks/configure_aap.yml.  

The following is the procedure to configure the AAP as the maint user.  

1. Login as the maint user
1. Edit /var/opt/aap_config/customer_vars.yml

   ```yaml
   ---
   ## Network Variables
   local_connection_name: enp1s0

   hostname: aap-apln

   # yaml list of dns servers
   dns_servers:
     - 192.168.122.1
     - 8.8.8.8

   domain: "example.com"
   address: 192.168.122.125/24
   gateway: 192.168.122.1
   time_server_list:
     - 0.rhel.pool.ntp.org
     - 1.rhel.pool.ntp.org
     - 2.rhel.pool.ntp.org
     - 3.rhel.pool.ntp.org

   ## AAP Installation Variables
   aap_password: changeme
   database_password: changeme
   vault_password_string: changeme
   ```

1. If your network has DHCP you should be able to use SCP/SFTP to transfer an AAP manifest.zip file to the AAP.  If your network does not have DHCP, you will need to run the configuration script to configure the network first.  It will fail with a file not found message for manifest.zip.  This is expected.

   ```shell
   scp manifest.zip maint@192.168.122.125:/var/opt/aap_config/manifest.zip
   ```

1. Run the following to configure the OS and Install AAP.

   ```shell
   sudo -u ansible /var/opt/run_config.bash
   ```

1. If you network does not have DHCP, the process will fail with file not found for manifest.zip.  Now upload your manifest zip using the above command, and re-run the script.  

Once the Process finishes, you will be able to login to the web interface of the AAP.  You can also use Config as Code to deploy a full configuration to the AAP at this time.  For instance, a config as code section can be added to /var/home/ansible/ansible_playbooks/configure_aap.yml to configure the entire system at install time.  
