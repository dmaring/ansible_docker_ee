# Provisioning a cluster with Docker EE, Docker Universal Control Plane (UCP) and Docker Trusted Registry (DTR) using Ansible

## Deploy a cluster

Ansible requires a cluster of booted and ready to use machines, accessible using [OpenSSH](https://www.openssh.com) (or [WinRM](https://msdn.microsoft.com/en-us/library/aa384426.aspx) for Windows). You can deploy your cluster manually or use deployment tools such as Terraform.

The cluster should be composed with freshly installed machines to ensure only Ansible is responsible of software provisioning. However, the [Playbook](http://docs.ansible.com/ansible/latest/playbooks.html) should be smart enough to also provision only what's needed.

You also need a Fully Qualified Domain Name (FQDN) pointed to the IP address of one of your managers or to a loadbalancer pointing on them.

Docker EE is only available on ([details](https://docs.docker.com/engine/installation/)):

* CentOS
* Ubuntu
* Windows Server
* Oracle Linux
* RHEL
* SLES

### Special instructions for Windows machine

To allow Ansible for deploying on Windows machines, a few steps are required. Instructions here: [http://docs.ansible.com/ansible/latest/intro_windows.html]().

Also, you need to install pywinrm (>=0.2.2) on the host machine ([see](http://docs.ansible.com/ansible/latest/intro_windows.html)).

## Terminology

Consult [http://docs.ansible.com/ansible/latest/glossary.html](http://docs.ansible.com/ansible/latest/glossary.html) for Ansible-specific terminology.

## Requirements

### Tools

- Ansible (2.4+): [http://docs.ansible.com/ansible/latest/intro_installation.html](http://docs.ansible.com/ansible/latest/intro_installation.html).

If you want to use Windows hosts, see [Ansible for Windows Hosts](http://docs.ansible.com/ansible/latest/intro_windows.html).

### OpenSSH

As mentioned earlier, Ansible use OpenSSH to connect to hosts, using key authentication. Hence, you need to generate a key-pair and add the public key in every single hosts' `authorized_keys` file (`/etc/ssh/authorized_keys` or `.ssh/authorized_keys`).

Ansible higly recommend to set up SSH agent to avoid retyping passwords, you can do:

```
$ ssh-add <path to the private key>`
```

See [ssh-keygen](https://man.openbsd.org/ssh-keygen), [ssh-agent](https://man.openbsd.org/ssh-agent) and [ssh-add](https://man.openbsd.org/ssh-add) for complementary information.

Host key checking (checking that the host key is present in your `known_hosts`) is enabled by default by Ansible and, the first time your run Ansible, you'll be prompted for everything single host if it's not already present in your `known_hosts`. For convenience, you can disable this checking by changing `host_key_checking` in `ansible.cfg`. However, it's not recommended in production.

## Inventory setup

The Docker Certified Infrastructure stacks produce a host file in the `inventory` directory.  If you need to create an inventory, edit `inventory/1.hosts` with the values of the following variables, according to your cluster and leave `inventory/2.groups` as is. The format for hosts is:

`<name-of-the-host> ansible_host=<ip-address>`

_N.B. The folder `example` contains template inventories showing specific use cases._

### Fill your groups

Groups to fill are listed under `inventory/1.hosts`:

| Group Name                    | Role    | Count | Recommended         |
|-------------------------------|---------|-------|---------------------|
| linux-ucp-managers-primary    | manager |   1   |          1*         |
| linux-ucp-managers-replicas   | manager |  >=0  |         2-4         |
| windows-ucp-managers-replicas | manager |  >=0  |         2-4         |
| linux-dtr-workers-primary     | worker  |   1   |          1*         |
| linux-dtr-workers-replicas    | worker  |  >=0  |         2-4         |
| windows-dtr-workers-replicas  | worker  |  >=0  |         2-4         |
| linux-workers                 | worker  |  >=0  | as many as you want |
| windows-workers               | worker  |  >=0  | as many as you want |

*: One and only one.

_N.B. If you are using an infrastructure provider (e.g. Terraform), make sure the inventory generated contains this groups._

> Files in the inventory are parsed in an alphabetical order. If generated by a third party program, make sure to always name the file containing the list of hosts `1.hosts`.

### Generated groups

A few groups are generated for convenience and used by the playbook.

| Group Name       | Composed by                                              |
|------------------|----------------------------------------------------------|
| ucp-primary      | linux-ucp-managers-primary                               |
| linux-ucp        | linux-ucp-managers-primary, linux-ucp-managers-replicas  |
| windows-ucp      | windows-ucp-managers-replicas                            |
| ucp              | linux-ucp, windows-ucp                                   |
| dtr-primary      | linux-dtr-workers-primary                                |
| dtr-replicas     | linux-dtr-workers-replicas, windows-dtr-workers-replicas |
| linux-dtr        | linux-dtr-workers-primary, linux-dtr-workers-replicas    |
| windows-dtr      | windows-dtr-workers-replicas                             |
| dtr              | linux-dtr, windows-dtr                                   |
| linux-managers   | linux-ucp-managers-primary, linux-ucp-managers-replicas  |
| windows-managers | windows-ucp-managers-replicas                            |
| linux-workers*   | linux-dtr                                                |
| windows-workers* | windows-dtr                                              |
| managers         | linux-managers, windows-managers                         |
| workers          | linux-workers, windows-workers                           |
| linux            | linux-managers, linux-workers                            |
| windows          | windows-managers, windows-workers                        |

*: Completed afterward by listed groups.

Do not edit the file `inventory/2.groups`

### Test your inventory

Assuming your inventory is `inventory/`, you can check that your groups are well-formed by running:

`ansible <name of the group> --list-hosts`

> We higly encourage you to check groups `managers`, `workers`, `ucp` and `dtr` before you go any further.

Then, a good way to check if hosts are reachable is pinging every single node in the cluster, which can be achieved by running:

`ansible linux -m ping all`

and

`ansible windows -m win_ping all`

_N.B. If python is not yet installed on the host, you can you the `raw` module, `ansible -i inventory linux -m raw -a "echo ping"`._

_N.B. You may need to provide `ansible_user` (the default ssh user name to use) for the ping and win_ping to work. See section Variables below for more information._

## Variables

Before you can run Ansible, you need to set a few variables.

Group-specific variables are located under `group_vars/<name of the group>`, a few placeholders have been set.

Star by running `grep -R placeholder` and edit the placeholders when it's needed. Variables with no default value are mandatory, if the specified group is used.

Group-specific variables are located under `group_vars/<name of the group>`, host-specific variables under `host_vars/<name of the host>`. A few placeholders have been set (run `grep -R placeholder`) to be changed and uncommented.

| Variables                        | Required by      | Location            | Default              |
|-------------------------------   |------------------|---------------------|----------------------|
| docker_ee_version                | all              | group_vars/all      | 17.06                |
| docker_ucp_version               | ucp              | group_vars/ucp      | latest               |
| docker_ee_subscription_ubuntu    | managers[Ubuntu] | group_vars/managers |                      |
| docker_ee_subscription_centos    | managers[CentOS] | group_vars/managers |                      |
| docker_ee_subscription_rhel      | managers[RHEL]   | group_vars/managers |                      |
| docker_ucp_admin_username        | ucp-primary, dtr | group_vars/all      | admin                |
| docker_ucp_admin_password        | ucp-primary, dtr | group_vars/all      |                      |
| docker_ucp_license               | ucp              | group_vars/ucp      \ license/Docker.lic   |
| docker_ucp_certificate_directory | ucp              | group_vars/ucp      \ ssl_cert             |
| docker_ucp_lb                    | ucp, dtr         | group_vars/all      | IP of first UCP node |
| docker_dtr_lb                    | dtr              | group_vars/dtr      | IP of DTR node       |
| docker_dtr_version               | dtr              | group_vars/dtr      | latest               |
| docker_dtr_replica_id            | dtr              | group_vars/dtr      |                      |
| cloudstor_plugin_options         | all	      | group_vars/all	    | (optional)           |

Variables can also be overwritten as follows:

`inventory/1.hosts`:
```ini
[windows-dtr-workers-primary]
windows-worker-1 ansible_host=<IP> ansible_password=<password>
```

Or by using `ansible-playbook [...] -e <variable_name>=<value>`.

For details about variables precedence, see [variable precedence in Ansible](http://docs.ansible.com/ansible/latest/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable).

## Fill licenses and certificates

### License

Start by creating a directory named `license/`.

Go to https://store.docker.com/my-content and download the license key file. Copy the file to `license/` as `Docker.lic`.

### Certificate

Start by creating a directory named `ssl_cert/`.

Get or generate a certificate for your `FQDN` and copy:

* The Server Certificate to `ssl_cert/` as `cert.pem`
* The Certificate Authority (CA) to `ssl_cert/` as `ca.pem` (For self-signed certificates, copy `cert.pem` as `ca.pem`).
* The private key of the certificate to `ssl_cert/` as `key.pem`

_N.B.: You can upload certificates and keys after, using the UCP GUI._

## Run Ansible

Ansible relies on `Playbooks`, describing a policy you want your remote systems to enforce, or a set of steps in a general IT process.

Ansible will iterate over tasks listed in `main.yml` and deploy:

* Docker EE across all nodes
* Docker Swarm across all nodes
* Docker UCP across all nodes
* Docker DTR across the selected nodes

Run `ansible-playbook main.yml` or `ansible-playbook main.yml [-e variable=value]`.


