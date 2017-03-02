# Open OnDemand

The Open OnDemand Project is an open-source software project, based on the Ohio Supercomputer Center's proven "OSC OnDemand" platform, that enables HPC centers to install and deploy advanced web and graphical interfaces for their users. More information can be found in the paper [http://dx.doi.org/10.1145/2949550.2949644](http://dx.doi.org/10.1145/2949550.2949644).

* [Section 1. Components](#section-1-components)
* [Section 2. Installation Guide](#section-2-installation-guide)
  * [2.1 - Software Requirements](#21---software-requirements)
  * [2.2 - Generate Apache Config for Open OnDemand Portal](#22---generate-apache-config-for-open-ondemand-portal)
  * [2.3 - Install Open OnDemand Proxy Module for Apache](#23---install-open-ondemand-proxy-module-for-apache)
  * [2.4 - Install the PUN Utility](#24---install-the-pun-utility)
  * [2.5 - Install User Mapping Script](#25---install-user-mapping-script)
  * [2.6 - Add Cluster Connection Config Files](#26---add-cluster-connection-config-files)
* [Section 3. System Apps](#section-3-system-apps)
    * [3.1 - Dashboard App](#31---dashboard-app)
    * [3.2 - Shell App](#32---shell-app)
    * [3.3 - Files App](#33---files-app)
    * [3.4 - File Editor App](#34---file-editor-app)
    * [3.5 - Active Jobs App](#35---active-jobs-app)
    * [3.6 - My Jobs App](#36---my-jobs-app)
    * [3.7 - User Documentation](#37---user-documentation)
* [Section 4. App Deployment Strategy](#section-4-app-deployment-strategy)
  * [4.1 - Local Directory Structure](#41---local-directory-structure)
  * [4.2 - Mapping URI to Local Directory Structure](#42---mapping-uri-to-local-directory-structure)
    * [4.2.1 - Apps URI](#421---apps-uri)
    * [4.2.2 - Public URI](#422---public-uri)
  * [4.3 - OSC Portals](#43---osc-portals)
    * [4.3.1 - Add/Update System Apps (requires root)](#431---addupdate-system-apps-requires-root)
    * [4.3.2 - Add/Update User Apps (requires root)](#432---addupdate-user-apps-requires-root)
    * [4.3.3 - Add/Update Dev Apps](#433---addupdate-dev-apps)
    * [4.3.4 - Add/Update Public Assets (requires root)](#434---addupdate-public-assets-requires-root)
* [Appendix A. CILogon Authentication Strategy (expert mode)](#appendix-a-cilogon-authentication-strategy-expert-mode)
  * [A.1 - Discovery Page](#a1---discovery-page)
  * [A.2 - Registration Page](#a2---registration-page)
  * [A.3 - Mapping Helper Scripts](#a3---mapping-helper-scripts)

## Section 1. Components

The components that make up the Open OnDemand **infrastructure** includes a proxy layer that all traffic passes through using the securely encrypted SSL protocol on port 443. The [Apache proxy](https://httpd.apache.org/) parses the URI and dynamically determines where to route the traffic to. In most cases the traffic will be routed to the per-user [NGINX](https://www.nginx.com/) (PUN) web server.

The PUN is described as an NGINX server instance running as a system-level user listening on a [Unix domain socket](https://en.wikipedia.org/wiki/Unix_domain_socket). File and directory permissions are used to restrict access to this Unix domain socket such that only the proxy daemon can communicate with the PUN.

| Component | Release | Description |
| --------- | ------- | ----------- |
| [ood-portal-generator](https://github.com/OSC/ood-portal-generator) | ![GitHub Release](https://img.shields.io/github/release/osc/ood-portal-generator.svg) | Generates an Open OnDemand portal config for an Apache server that defines the proxy interface. |
| [mod_ood_proxy](https://github.com/OSC/mod_ood_proxy) | ![GitHub Release](https://img.shields.io/github/release/osc/mod_ood_proxy.svg) | An Apache httpd module implementing the Open OnDemand proxy API. |
| [ood_auth_map](https://github.com/OSC/ood_auth_map) | ![GitHub Release](https://img.shields.io/github/release/osc/ood_auth_map.svg) | The user mapping script that maps the authenticated username to the system-level username. |
| [nginx_stage](https://github.com/OSC/nginx_stage) | ![GitHub Release](https://img.shields.io/github/release/osc/nginx_stage.svg) | Stages and controls the per-user NGINX (PUN) instances. |

## Section 2. Installation Guide

This installation tutorial starts with a web host webdev05.osc.edu which has

* TORQUE client libraries installed and TORQUE authd daemon started
* SSL public and private certificates and a chain certificate
* `scl`, `lsof`, and `sudo` are installed
* The home directory file system is mounted and accessible by users
* The host has been added as submit hosts to the servers it will be submitting jobs to
* **FIXME: is there anything else I missed?**

### 2.1 - Software Requirements

We will use RedHat Software Collections to satisfy these requirements:

* Apache HTTP Server 2.4
* NGINX 1.6
* Phusion Passenger 4.0
* Ruby 2.2 with rake, bundler, and ability to build gem native extensions
* Node.js 0.10
* Git 1.9

In this tutorial scl (RedHat Software Collections) packages happen to be installed at `/opt/rh`.
This tutorial is also done from an account that has sudo access but is not root.

Enable the SCL location to pull rpms from:

```
sudo subscription-manager repos --enable=rhel-server-rhscl-6-rpms
# => Repository 'rhel-server-rhscl-6-rpms' is enabled for this system.
```

Install dependencies:

```
sudo yum install -y httpd24 nginx16 rh-passenger40 rh-ruby22 rh-ruby22-rubygem-rake rh-ruby22-rubygem-bundler rh-ruby22-ruby-devel nodejs010 git19
```

Update Apache Environment to include rh-ruby22. We need this for the user mapping
script which is written in ruby. Do this by editing `/opt/rh/httpd24/service-environment`:

```diff
-HTTPD24_HTTPD_SCLS_ENABLED="httpd24"
+HTTPD24_HTTPD_SCLS_ENABLED="httpd24 rh-ruby22"
```

Finally, make src directory where we will check out and build OOD infrastructure components and apps:

```sh
mkdir -p ~/tmp/ood/src
cd ~/tmp/ood/src
```


### 2.2 - Generate Apache Config

1. Clone and check out the latest tag:

    ```sh
    scl enable git19 -- git clone https://github.com/OSC/ood-portal-generator.git
    cd ood-portal-generator/
    git checkout 0.3.0
    ```

2. `ood-portal-generator` is a script that takes a config.yml (or uses defaults if not provided) and renders an Apache config template file. Generate a default one now:

    ```sh
    scl enable rh-ruby22 -- rake
    # => mkdir -p build
    # => rendering templates/ood-portal.conf.erb => build/ood-portal.conf
    ```

3. Copy this to the default installation location:

    ```
    sudo scl enable rh-ruby22 -- rake install
    # => cp build/ood-portal.conf /opt/rh/httpd24/root/etc/httpd/conf.d/ood-portal.conf
    ```

4. For now, lets use basic auth with an .htpasswd file till we get the installation complete.
Then we will add another authentication mechanism.

    ```sh
    sudo scl enable httpd24 -- htpasswd -c /opt/rh/httpd24/root/etc/httpd/.htpasswd efranz
    #=> New password:
    #=> Re-type new password:
    #=> Adding password for user efranz
    ```

_Note: The Apache config references the location of `mod_ood_proxy`, `nginx_stage`, and `ood_auth_map`. Be sure to update these locations if you change the `PREFIX` for any installation of the corresponding package in the config.yml prior to generating the Apache config._

### 2.3 - Install Open OnDemand Proxy Module for Apache

An Apache module written in Lua is the primary component for the proxy logic. It is given by the [mod_ood_proxy](https://github.com/OSC/mod_ood_proxy) project.

1.  We clone the `mod_ood_proxy` repo to the local disk:

    ```sh
    git clone https://github.com/OSC/mod_ood_proxy.git
    cd mod_ood_proxy
    ```

2.  Check out the version of the generator you intend on using:

    ![GitHub Release](https://img.shields.io/github/release/osc/mod_ood_proxy.svg?label=latest release)

    ```sh
    # A list of versions & details can be viewed in the CHANGELOG.md
    # As of writing this the latest version is `v0.2.0`
    git checkout tags/v0.2.0
    ```

3.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => /opt/ood/mod_ood_proxy
    ```

### 2.4 - Install the PUN Utility

The PUNs are manipulated and maintained by the [nginx_stage](https://github.com/OSC/nginx_stage) utility. This tool is meant to by run by `root` or a user with `sudoers` privileges.

1.  We clone the `nginx_stage` repo to the local disk:

    ```sh
    git clone https://github.com/OSC/nginx_stage.git
    cd nginx_stage
    ```

2.  Check out the version of the generator you intend on using:

    ![GitHub Release](https://img.shields.io/github/release/osc/nginx_stage.svg?label=latest release)

    ```sh
    # A list of versions & details can be viewed in the CHANGELOG.md
    # As of writing this the latest version is `v0.2.0`
    git checkout tags/v0.2.0
    ```

3.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => /opt/ood/nginx_stage
    ```

4.  Modify `/opt/ood/nginx_stage/config/nginx_stage.yml` and `/opt/ood/nginx_stage/bin/ood_ruby`.

    In particular, **opt into** metrics reporting in `nginx_stage.yml` and use Software Collections in `ood_ruby`.

    Note: These files will not be overwritten the next time you update `nginx_stage`.
    
    **Pro tip**: If you run an older Linux OS that creates user accounts starting at id `500`, then you will need to modify the configuration option `min_uid: 1000` accordingly.

5.  If not already done, give the `apache` user `sudo` privileges to run this tool:

    ```sh
    # /etc/sudoers

    Defaults:apache     !requiretty, !authenticate
    apache ALL=(ALL) NOPASSWD: /opt/ood/nginx_stage/sbin/nginx_stage
    ```

### 2.5 - Install User Mapping Script

You will need to map the Apache authenticated user to the local system user. This is done with the simple tool: [ood_auth_map](https://github.com/OSC/ood_auth_map).

1.  We clone the `ood_auth_map` repo to the local disk:

    ```sh
    git clone https://github.com/OSC/ood_auth_map.git
    cd ood_auth_map
    ```

2.  Check out the version of the generator you intend on using:

    ![GitHub Release](https://img.shields.io/github/release/osc/ood_auth_map.svg?label=latest release)

    ```sh
    # A list of versions & details can be viewed in the CHANGELOG.md
    # As of writing this the latest version is `v0.0.3`
    git checkout tags/v0.0.3
    ```

3.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => /opt/ood/ood_auth_map
    ```

The principle behind this script is that you call it with a URL encoded `REMOTE_USER` user name as the only argument, and it will return the mapping to the local system user name if it exists.

### 2.6 - Add Cluster Connection Config Files

**(Optional step)**

The Dashboard, File Explorer, and Shell Access can work without cluster
connection config files. These config files are required for:

* enable Shell Access to multiple named hosts outside of the local host OOD is running on
* use Active Jobs, My Jobs, or any other app that works with batch jobs


1. Create default directory for config files:

    ```sh
    sudo mkdir -p /etc/ood/config/clusters.d
    ```

2. Add one config file for each host you want to provide access to. Each config file is a YAML file and must have the `.yml` suffix.

Here is the minimal YAML config required to specify a host that can be accessed via Shell Access app. The filename is `oakley.yml`:

```yaml
---
v1:
  title: "Oakley"
  cluster:
    type: "OodCluster::Cluster"
    data:
      servers:
        login:
          type: "OodCluster::Servers::Ssh"
          data:
            host: "oakely.osc.edu"
```

* a cluster contains one ore more servers, with server names "login", "resource_mgr" and "scheduler"
* special server keywards are:
  * login
  * resource_mgr
  * scheduler
  * ganglia

For Active Jobs and My Jobs to work, a cluster configuration must specify a resource_mgr to use.

```yaml
---
v1:
  title: "Oakley"
  cluster:
    type: "OodCluster::Cluster"
    data:
      servers:
        login:
          type: "OodCluster::Servers::Ssh"
          data:
            host: "oakely.osc.edu"
        resource_mgr:
          type: "OodCluster::Servers::Torque"
          data:
            host: "oak-batch.osc.edu"
            lib: "/opt/torque/lib64"
            bin: "/opt/torque/bin"
            version: "6.0.1"
```

The name of the file becomes the key for this host. So `oakley.yml` cluster config will have a key `oakley`. My Jobs and other OOD apps that cache information about jobs they manage will associate job metadata with this key.

## Section 3. System Apps

These are the apps deployed by the system administrator on the local disk that are accessible by all users.

### 3.1 - Dashboard App

See https://github.com/OSC/ood-dashboard for more information.

### 3.2 - Shell App

See https://github.com/OSC/ood-shell for more information.

### 3.3 - Files App

See https://github.com/OSC/ood-fileexplorer for more information.

### 3.4 - File Editor App

See https://github.com/OSC/ood-fileeditor for more information.

### 3.5 - Active Jobs App

See https://github.com/OSC/ood-activejobs for more information.

### 3.6 - My Jobs App

See https://github.com/OSC/ood-myjobs for more information.

### 3.7 - User Documentation

Currently there is no general user documentation provided.

OSC provides user documentation for their Open OnDemand deployment (OSC OnDemand) at:

https://www.osc.edu/resources/online_portals/ondemand

Feel free to use it as template for your own organization's user-facing documentation.

## Section 4. App Deployment Strategy

This is the strategy currently employed at the OSC OnDemand and OSC AweSim portals for deploying apps. This is in no way a requirement, and system administrators are encouraged to explore different options.

### 4.1 - Local Directory Structure

Apps are deployed on the local disk of the host serving the Open OnDemand portal. Care must be taken for user shared apps as they are deployed through symlinks to the corresponding user's home directory on the shared file system.

```sh
/var/www/ood                                       # drwxr-xr-x  root    root
├── apps                                           # drwxr-xr-x  root    root
│   ├── sys                                        # drwxr-xr-x  root    root
│   │   ├── <APP>                                  # drwxr-xr-x  root    root
│   │   │   ├── config.ru                          # -rw-r--r--  root    root
│   │   │   └── ...                                # -rw-r--r--  root    root
│   │   ├── dashboard                              # drwxr-xr-x  root    root
│   │   │   ├── config.ru                          # -rw-r--r--  root    root
│   │   │   └── ...                                # -rw-r--r--  root    root
│   │   └── ...                                    # drwxr-xr-x  root    root
│   └── usr                                        # drwxr-xr-x  root    root
│       ├── <USER>                                 # drwxr-x---  root    <GROUP>
│       │   └── gateway -> ~<USER>/<PORTAL>/share  # lrwxrwxrwx  root    root
│       │       ├── <APP>                          # drwx------+ <USER>  <GROUP>
│       │       │   ├── config.ru                  # -rw-r--r--  <USER>  <GROUP>
│       │       │   └── ...                        # -rw-r--r--  <USER>  <GROUP>
│       │       └── ...                            # drwx------+ <USER>  <GROUP>
│       └── efranz                                 # drwxr-x---  root    PZS0562
│           └── gateway -> ~efranz/<PORTAL>/share  # lrwxrwxrwx  root    root
│               ├── activejobs                     # drwx------+ efranz  PZS0562
│               │   ├── config.ru                  # -rw-r--r--  efranz  PZS0562
│               │   └── ...                        # -rw-r--r--  efranz  PZS0562
│               └── ...                            # drwx------+ efranz  PZS0562
├── discover                                       # drwxr-xr-x  root    root
│   ├── index.php                                  # -rw-r--r--  root    root
│   └── ...                                        # -rw-r--r--  root    root
├── public                                         # drwxr-xr-x  root    root
│   ├── favicon.ico                                # -rw-r--r--  root    root
│   └── ...                                        # -rw-r--r--  root    root
└── register                                       # drwxr-xr-x  root    root
    ├── index.php                                  # -rw-r--r--  root    root
    └── ...                                        # -rw-r--r--  root    root
```

### 4.2 - Mapping URI to Local Directory Structure

#### 4.2.1 - Apps URI

To access an app you **MUST**:

  - be authenticated through Apache (in particular authenticated through CILogon)
  - have authenticated user be mapped to a local system user
  - have `uid > 1000`
  - have a valid shell

```sh
# Accessing a system app
https://<PORTAL>.osc.edu/pun/sys/<APP>         =>  /var/www/ood/apps/sys/<APP>

# Accessing a user shared app
https://<PORTAL>.osc.edu/pun/usr/<USER>/<APP>  =>  /var/www/ood/apps/usr/<USER>/gateway/<APP>
                                               =>  ~<USER>/<PORTAL>/share/<APP>

# Accessing your own sandbox app
https://<PORTAL>.osc.edu/pun/dev/<APP>         =>  ~/<PORTAL>/dev/<APP>
```

#### 4.2.2 - Public URI

**Anyone** can access the resources underneath the `/public` URI.

```sh
# Accessing a publicly available resource
https://<PORTAL>.osc.edu/public/favicon.ico  =>  /var/www/ood/public/favicon.ico
```

### 4.3 - OSC Portals

Available OSC Portals:

| `<PORTAL>` | `<HOST>` |
| ---------- | -------- |
| `ondemand` | `web02.hpc.osc.edu` |
| `awesim`   | `web04.hpc.osc.edu` |

#### 4.3.1 - Add/Update System Apps (requires root)

The system apps are maintained by mirroring the following directory:

```
/users/PZS0645/wiag/ood_portals/<PORTAL>/sys
```

The command to mirror this directory (**performed** on `webXX`) is:

```sh
# Mirror the deployment directory with the staged directory
sudo rsync -rlptvu --delete /users/PZS0645/wiag/ood_portals/<PORTAL>/sys/ /var/www/ood/apps/sys
```

#### 4.3.2 - Add/Update User Apps (requires root)

Each `<USER>` directory on the local disk will be given an appropriate permission such that only members of the `<GROUP>` can access the user's shared apps. This is an important security condition so that users don't access malicious developers' apps.

| `<PORTAL>` | `<GROUP>` |
| ---------- | --------- |
| `ondemand` | `<USER>`'s primary group |
| `awesim`   | `awesim`  |

To register a developer with username `<USER>` and groupname `<GROUP>` (**performed** on `webXX`):

```sh
# Create the user's app sharing directory
sudo mkdir -p /var/www/ood/apps/usr/<USER>

# Set proper permissions so only primary group members can access shared apps
# NB: This is important or any user can access this developer's apps which can be a security risk
sudo chmod 750 /var/www/ood/apps/usr/<USER>
sudo chgrp <GROUP> /var/www/ood/apps/usr/<USER>

# Create symlink to user's shared apps location in their homedir for given portal
sudo ln -s ~<USER>/<PORTAL>/share /var/www/ood/apps/usr/<USER>/gateway

# Optional: May need to create the directory for the user in their home directory
sudo -u <USER> mkdir -p ~<USER>/<PORTAL>/share
```

Now the user can update/maintain his or her own apps within their home directory:

```sh
~/<PORTAL>/share/*
```

#### 4.3.3 - Add/Update Dev Apps

The user will need to create the required dev directory in their home directory in order to develop apps on the given `<PORTAL>`:

```sh
# Create directory that holds dev apps
mkdir -p ~/<PORTAL>/dev
```

#### 4.3.4 - Add/Update Public Assets (requires root)

The public assets are maintained by mirroring the following directory:

```
/users/PZS0645/wiag/ood_portals/<PORTAL>/public
```

The command to mirror this directory (**performed** on `webXX`) is:

```sh
# Mirror the deployment directory with the staged directory
sudo rsync -rlptvu --delete /users/PZS0645/wiag/ood_portals/<PORTAL>/public/ /var/www/ood/public
```

## Appendix A. CILogon Authentication Strategy (expert mode)

**Warning**: Below is not for the faint of heart. It is missing documentation and can be very difficult to fully grasp. It is not recommended to deploy a CILogon authentication strategy on your own. You have been warned!

The current strategy employed by the OSC OnDemand portal is outlined in detail
in the figure below.

[![Authentication Strategy](http://www.plantuml.com/plantuml/png/jLPDRzim3BtxL_3OO4WSTBS0MrgimL2WMuEammvBKIWMumXp94rIqcQ_VfIR3tkMQwzPJpPaaU_naNpmXNNCkFKgYrZYb87BJ2GOQJeJYT1CElYEyw-Abyy-NT-eLCdIsJKVHr4U3jsF-wW1V1xTmT8vRGTnl5JMnKnhZoKspE4X-UvPYbGQfU09MBHMA3cJ-2Ii6yAPL9rZ04Nae0FuyRv_VWyJvC9WuigWNDX4RT1472lORKWVukkP5uYSz_ltSBKBsSAOfVXQO4KAn4DXxOTUhzSAVOhkA9tpbnEbVWgSoHg14f7vPlQKIMEsTajbn6ySUhWNEUzceCovFjSXqtvsTi-gS3H8S4EThkYsZmnm6DzE6qZ0sLpnxI3Fdf2qA3ljE8L54rntymRzBLIL924EW0w5X7SmHhCI3bYIq5GV2VZbaOfUCbnG-nRFGQjrveJEq4iS-n9dqk2lDLfd8LL0H2dd4Wr6lgfcqaLXKdHhYwR2lxp6ZKqkfJb1ps3FMZQ_SG1F8LROmiPE1zVu4UUI3WivMhbVOj1VOKVxMnpiER3rLvWXNGylViaIdjCrcSB1gb_fMwUx_1Pj9nZXvmrM181yMCN8jyZxxZ60xi9azENsRlJmRJgQW5DWjzRXwi6dciapUa1g1AUBtON8i1CfxK2g74N851PgBNa44A8Z5Nq8g5IutMx83APKEPgXyPzplwz9NDM_ET5B_5gt8rnJxjgJ7DGmvuShT-xMv4lcNLF1thvznoHROQoPjO_AyCuRr5OSDirzOZyTNPe6FnqpAHQmOPdWVaavIYTOPl-zttv5MciKNB2thRouPa6a_xDgI9iLmYTg8VC9NTL6FMqzbXUbTaBf0rngueTMAsc3lsGNn8P-Xly0)](http://www.plantuml.com/plantuml/uml/jLPDRzim3BtxL_3OO4WSTBS0MrgimL2WMuEammvBKIWMumXp94rIqcQ_VfIR3tkMQwzPJpPaaU_naNpmXNNCkFKgYrZYb87BJ2GOQJeJYT1CElYEyw-Abyy-NT-eLCdIsJKVHr4U3jsF-wW1V1xTmT8vRGTnl5JMnKnhZoKspE4X-UvPYbGQfU09MBHMA3cJ-2Ii6yAPL9rZ04Nae0FuyRv_VWyJvC9WuigWNDX4RT1472lORKWVukkP5uYSz_ltSBKBsSAOfVXQO4KAn4DXxOTUhzSAVOhkA9tpbnEbVWgSoHg14f7vPlQKIMEsTajbn6ySUhWNEUzceCovFjSXqtvsTi-gS3H8S4EThkYsZmnm6DzE6qZ0sLpnxI3Fdf2qA3ljE8L54rntymRzBLIL924EW0w5X7SmHhCI3bYIq5GV2VZbaOfUCbnG-nRFGQjrveJEq4iS-n9dqk2lDLfd8LL0H2dd4Wr6lgfcqaLXKdHhYwR2lxp6ZKqkfJb1ps3FMZQ_SG1F8LROmiPE1zVu4UUI3WivMhbVOj1VOKVxMnpiER3rLvWXNGylViaIdjCrcSB1gb_fMwUx_1Pj9nZXvmrM181yMCN8jyZxxZ60xi9azENsRlJmRJgQW5DWjzRXwi6dciapUa1g1AUBtON8i1CfxK2g74N851PgBNa44A8Z5Nq8g5IutMx83APKEPgXyPzplwz9NDM_ET5B_5gt8rnJxjgJ7DGmvuShT-xMv4lcNLF1thvznoHROQoPjO_AyCuRr5OSDirzOZyTNPe6FnqpAHQmOPdWVaavIYTOPl-zttv5MciKNB2thRouPa6a_xDgI9iLmYTg8VC9NTL6FMqzbXUbTaBf0rngueTMAsc3lsGNn8P-Xly0)

The workflow for an unauthenticated user accessing a protected resource behind
the Apache proxy can be briefly described as:

1.  The Apache proxy redirects the user to the
    [ood_discovery](https://github.com/OSC/ood_auth_discovery/) page
2.  The user then clicks the link to the CILogon authentication portal
3.  After choosing their authentication provider and successfully
    authenticating
4.  If a mapping exists for the authenticated user to a local system user:

    1. User is given access to the protected resource

    If the mapping doesn't exist:

    1. The user is redirected to the
       [ood_registration](https://github.com/OSC/ood_auth_registration/) page
    2. The user registers their authenticated user name to a local system user
       name by authenticating against a local LDAP server
    3. The mapping is generated if successfully authenticated
    4. The user is again redirected to the protected resource

### A.1 - Discovery Page

Before a user is authenticated, the user is presented with a discovery page where he/she can choose the OpenID Connect Provider. For the [ood_auth_discovery](https://github.com/OSC/ood_auth_discovery) repo it is a branded page with a link to CILogon.

1.  We clone the `ood_auth_discovery` repo to the local disk:

    ```sh
    git clone https://github.com/OSC/ood_auth_discovery.git
    ```

2.  Install it to its global location:

    ```sh
    # create the /var/www/ood directory if it doesn't already exist
    sudo mkdir -p /var/www/ood

    # copy this repo to its default location
    sudo cp -r ood_auth_discovery /var/www/ood/discover
    ```

**Anyone** can access the resources underneath the `/discover` URI.

```sh
# Accessing the discovery page
https://<PORTAL>.osc.edu/discover  =>  /var/www/ood/discover/index.php
```

### A.2 - Registration Page

After a user is authenticated and it is determined that no mapping exists to a local system user, they are redirected to the [ood_auth_registration](https://github.com/OSC/ood_auth_registration) branded page. Here the user is required to enter their local system credentials and the mapping is generated.

1.  We clone the `ood_auth_registration` repo to the local disk:

    ```sh
    git clone https://github.com/OSC/ood_auth_registration.git
    ```

2.  Install it to its global location:

    ```sh
    # create the /var/www/ood directory if it doesn't already exist
    sudo mkdir -p /var/www/ood

    # copy this repo to its default location
    sudo cp -r ood_auth_registration /var/www/ood/register
    ```

3.  This repo makes use of a securely encrypted `ldaps://...` server for authenticating a user to the local system. This requires the appropriate OpenLDAP Certificate Authority files be hosted on the local machine:

    ```sh
    /etc/openldap/cacerts/*
    ```

4.  This repo also generates a proper user mapping after the user authenticates successfully on the local system. So the `apache` user will need permissions to generate this mapping on behalf of the user. This requires granting the `apache` user `sudo` privileges to the respective `mapdn` script.

    ```sh
    # /etc/sudoers

    apache      ALL=(ALL) NOPASSWD:  /usr/local/bin/add-user-dn.real, \
                                     /usr/local/bin/delete-user-dn.real, \
                                     /usr/local/bin/list-user-dns.real
    ```

To access any resource underneath the `/register` URI you **MUST**:

  - be authenticated through Apache (in particular authenticated through CILogon)

```sh
# Accessing the registration page
https://<PORTAL>.osc.edu/register  =>  /var/www/ood/register/index.php
```

### A.3 - Mapping Helper Scripts

**FIXME**

[ood_auth_mapdn](https://github.com/OSC/ood_auth_mapdn)


