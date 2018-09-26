# Ansible-Junos-Skeleton

This project contains a sample Ansible project and it can be used with any Ansible installation.

> Currently confirmed with Ansible 2.4

For ease, I recommend this is used with the following Docker container.

```bash
docker pull hub.docker.com/davidjohngee/juniper-gitlab-container:latest
```

Clone this repository in to a directory and then mount the directory as a volume in to the Docker container. That way the container can work directly with the contents of the project.

This bash command will start the container and mount the volume in one fell swoop.

> Interesting note (perhaps): The container can be used as a GitLab runner hence the name. 

```bash
docker run -it --name myansibletest --volume /PATH/TO/DIR:/project davidjohngee/juniper-gitlab-container bash

root@582d8b3b759b:/project#
```

Note, the prompt for me will be different to the prompt for you. Expect the ID to be a different number. The pattern however should remain the same.

Now we're cooking! The next job is to add a Junos node to your container through Ansible configuration and the `/etc/hosts` file if your node doesn't have a DNS FQDN reference.

> Note, my `vsrx` management interface has IP address: 192.168.10.66.

```bash
vim /etc/hosts
# Now edit the file and add your line. For instance mine would be:
vsrx01 192.168.10.66
# Hit `escape` and then `:wq!` to write and exit vim simultaneously.
```

Now the editing is done, I should be able to ping the device through the normal method:

```bash
root@582d8b3b759b:/project# ping vsrx01
PING vsrx01 (192.168.10.66) 56(84) bytes of data.
64 bytes from vsrx01 (192.168.10.66): icmp_seq=1 ttl=37 time=1.35 ms
64 bytes from vsrx01 (192.168.10.66): icmp_seq=2 ttl=37 time=0.997 ms
64 bytes from vsrx01 (192.168.10.66): icmp_seq=3 ttl=37 time=0.905 ms
#... etc
```

Next task is to edit the Ansible hosts file.

```bash
vim hosts
# Now edit the hosts file
##############################################
# lab inventory
##############################################

vsrx01  junos_host=192.168.10.66
```

We're getting close to being able to test.

Our next step is to generate an SSH RSA key-pair and upload the public key to the vSRX. Be sure to set the hostname on the vSRX else outputs may not be named correctly from Ansible.

```bash
# Container
ssh-keygen
# Hit enter through all of the choices unless you want to password protect the SSH key. If you do password protect, ensure you remember the password as it will need to be part of your Ansible configuration. The password field will be used to unlock the key when being used.
cat ~/.ssh/id_rsa.pub
# Copy the output. This is your public key.
```

Login to the management interface on Junos via CLI over SSH.

```bash
# Junos
set system hostname vsrx01
set system login user ansible authentication ssh-rsa "<PASTE_PUBLIC_KEY>"
```

Assuming that the full path to the SSH key is as per this README file, then the `host_vars` file under `vsrx01` will not need modifying. If your host is not called `vsrx01` then you will need to create a directory under `host_vars` or create a file `provider_info` under `group_vars`. Copy the contents of `credentials.yml` into the new host_var file or group_vars.

Here is the snippet from the vsrx01 file under host_vars.

```bash
# credentials.yml
provider_info:
  host: "{{ inventory_hostname }}"
  user: ansible
  ssh_keyfile: "~/.ssh/id_rsa"
```

There is a strategy that the Ansible_Provider info can live in `group_vars` but if you want to use different SSH keys for different nodes, this strategy means `host_vars` will override the `group_vars`.

In the test playbook `pb.check_connect.yml`, the provider_info dictionary is used as per below.


```bash
---
- name: Get Device Facts
  hosts: vsrx01
  roles:
    - Juniper.junos
  connection: local
  gather_facts: no

  tasks:
    - name: Retrieve facts from devices running Junos OS
      juniper_junos_facts:
        provider: "{{ provider_info }}"
        savedir: "{{ playbook_dir }}"
    - name: Print version
      debug:
        var: junos.version
```

Assuming you've taken care of the host changes or are running with defaults on `vsrx01` naming (and you've changed the IP address suitably), we're now in a position to test! How exciting.

```bash
root@582d8b3b759b:/project# ansible-playbook pb.check_connect.yml

PLAY [Get Device Facts] *********************************************************************************************************************************************************************************************************************

TASK [Retrieve facts from devices running Junos OS] *****************************************************************************************************************************************************************************************
ok: [vsrx01]

TASK [Print version] ************************************************************************************************************************************************************************************************************************
ok: [vsrx01] => {
    "junos.version": "15.1X49-D140.2"
}

PLAY RECAP **********************************************************************************************************************************************************************************************************************************
vsrx01                     : ok=2    changed=0    unreachable=0    failed=0
```

And there have it. Two new files are also created in the project directory.

```bash
total 36
drwxr-xr-x 12 root root  384 Sep 26 16:28 .
drwxr-xr-x  1 root root 4096 Sep 26 09:52 ..
drwxr-xr-x 10 root root  320 Sep 26 16:12 .git
-rw-r--r--  1 root root   22 Sep 26 16:04 .gitignore
-rw-r--r--  1 root root   25 Sep 26 16:03 README.md
-rw-r--r--  1 root root  112 Sep 25 15:36 ansible.cfg
drwxr-xr-x  3 root root   96 Sep 25 15:36 host_vars
-rw-r--r--  1 root root  143 Sep 25 15:36 hosts
-rw-r--r--  1 root root    7 Sep 26 16:28 pb.check_connect.retry
-rw-r--r--  1 root root  353 Sep 25 15:36 pb.check_connect.yml
-rw-r--r--  1 root root 1419 Sep 26 16:29 vsrx01-facts.json # < New
-rw-r--r--  1 root root  784 Sep 26 16:29 vsrx01-inventory.xml # < New
```

Assuming you have `jq` installed, you can easily inspect the JSON.

```bash
root@582d8b3b759b:/project# cat vsrx01-facts.json | jq
{
  "domain": null,
  "hostname_info": {
    "re0": "vsrx01"
  },
  "version_RE1": null,
  "version_RE0": "15.1X49-D140.2",
  "re_master": {
    "default": "0"
  },
  "serialnumber": "3A8D534D7B77",
  "vc_master": null,
  "RE_hw_mi": false,
  "HOME": "/var/home/ansible",
  "master_state": true,
  "re_info": {
    "default": {
      "default": {
        "status": "OK",
        "last_reboot_reason": "0x4000:VJUNOS reboot",
        "model": "VSRX-S",
        "mastership_state": "master"
      },
      "0": {
        "status": "OK",
        "last_reboot_reason": "0x4000:VJUNOS reboot",
        "model": "VSRX-S",
        "mastership_state": "master"
      }
    }
  },
  "srx_cluster_id": null,
  "hostname": "vsrx01",
  "virtual": null,
  "version": "15.1X49-D140.2",
  "master": "RE0",
  "vc_fabric": null,
  "personality": null,
  "srx_cluster_redundancy_group": null,
  "version_info": {
    "major": [
      15,
      1
    ],
    "type": "X",
    "build": 2,
    "minor": [
      49,
      "D",
      140
    ]
  },
  "re_name": "re0",
  "srx_cluster": false,
  "vc_mode": null,
  "vc_capable": false,
  "ifd_style": "CLASSIC",
  "model_info": {
    "re0": "VSRX"
  },
  "RE0": {
    "status": "OK",
    "last_reboot_reason": "0x4000:VJUNOS reboot",
    "model": "VSRX-S",
    "up_time": "1 day, 2 hours, 43 minutes, 49 seconds",
    "mastership_state": "master"
  },
  "RE1": null,
  "fqdn": "vsrx01",
  "junos_info": {
    "re0": {
      "text": "15.1X49-D140.2",
      "object": {
        "major": [
          15,
          1
        ],
        "type": "X",
        "build": 2,
        "minor": [
          49,
          "D",
          140
        ]
      }
    }
  },
  "has_2RE": false,
  "switch_style": "VLAN_L2NG",
  "model": "VSRX",
  "current_re": [
    "master",
    "node",
    "fwdd",
    "member",
    "pfem",
    "fpc0",
    "re0",
    "fpc0.pic0"
  ]
}
```

You should be able to run any Ansible playbook now this basic setup is in place. Good luck!