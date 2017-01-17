# Open OnDemand

The Open OnDemand Project is an open-source software project, based on the Ohio Supercomputer Center's proven "OSC OnDemand" platform, that enables HPC centers to install and deploy advanced web and graphical interfaces for their users. More information can be found in the paper [http://dx.doi.org/10.1145/2949550.2949644](http://dx.doi.org/10.1145/2949550.2949644).

* [Section 1. Components](#section-1-components)
* [Section 2. Installation Guide](#section-2-installation-guide)
  * [2.1 - Software Requirements](#21---software-requirements)
  * [2.2 - Generate Apache Config for Open OnDemand Portal](#22---generate-apache-config-for-open-ondemand-portal)
  * [2.3 - Install Open OnDemand Proxy Module for Apache](#23---install-open-ondemand-proxy-module-for-apache)
  * [2.4 - Install the PUN Utility](#24---install-the-pun-utility)
  * [2.5 - Install User Mapping Script](#25---install-user-mapping-script)
* [Section 3. App Deployment Strategy](#section-3-app-deployment-strategy)
  * [3.1 - Local Directory Structure](#31---local-directory-structure)
  * [3.2 - Mapping URI to Local Directory Structure](#32---mapping-uri-to-local-directory-structure)
    * [3.2.1 - Apps URI](#321---apps-uri)
    * [3.2.2 - Public URI](#322---public-uri)
  * [3.3 - OSC Portals](#33---osc-portals)
    * [3.3.1 - Add/Update System Apps (requires root)](#331---addupdate-system-apps-requires-root)
    * [3.3.2 - Add/Update User Apps (requires root)](#332---addupdate-user-apps-requires-root)
    * [3.3.3 - Add/Update Dev Apps](#333---addupdate-dev-apps)
    * [3.3.4 - Add/Update Public Assets (requires root)](#334---addupdate-public-assets-requires-root)
* [Section 4. System Apps](#section-4-system-apps)
    * [4.1 - Dashboard App](#41---dashboard-app)
    * [4.2 - Shell App](#42---shell-app)
    * [4.3 - Files App](#43---files-app)
    * [4.4 - File Editor App](#44---file-editor-app)
    * [4.5 - User Documentation](#45---user-documentation)
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

### 2.1 - Software Requirements

**Assumes you are using RedHat Software Collections**

```
yum install -y centos-release-scl lsof sudo
yum install -y httpd24 nginx16 rh-passenger40 rh-ruby22 rh-ruby22-rubygem-rake rh-ruby22-rubygem-bundler rh-ruby22-ruby-devel nodejs010 git19
```

**Work in Progress**

### 2.2 - Generate Apache Config for Open OnDemand Portal

In this section we will generate an Open OnDemand Portal config file used by the Apache server. This can be done manually or using the [ood-portal-generator](https://github.com/OSC/ood-portal-generator).

1.  We clone the `ood-portal-generator` repo to the local disk:

    ```sh
    git clone https://github.com/OSC/ood-portal-generator.git
    cd ood-portal-generator
    ```

2.  Check out the version of the generator you intend on using:

    ![GitHub Release](https://img.shields.io/github/release/osc/ood-portal-generator.svg?label=latest release)

    ```sh
    # A list of versions & details can be viewed in the CHANGELOG.md
    # As of writing this the latest version is `v0.0.7`
    git checkout tags/v0.0.7
    ```

3.  Now we build the Apache config using environment variables to modify any of the default settings:

    ```sh
    scl enable rh-ruby22 -- rake
    ```

    Documentation on the available options can be found at:

    https://github.com/OSC/ood-portal-generator#default-options

    **SUBDOMAIN** -- If a subdomain or different IP address is used (assuming the SSL certificates are setup as well for this subdomain):

    ```sh
    scl enable rh-ruby22 -- rake OOD_IP=<ip> OOD_SERVER_NAME=<subdomain>

    # Example:
    scl enable rh-ruby22 -- rake OOD_IP='xxx.xxx.xxx.xxx' OOD_SERVER_NAME='ondemand.osc.edu'
    ```
    
    **Pro-tip**: For a *production* server we'd greatly appreciate it if you contributed analytics back to us for continued support of the project.
    
    ```sh
    scl enable rh-ruby22 -- rake OOD_ANALYTICS_OPT_IN=true
    ```

4.  Confirm the Apache config is to your liking by viewing the config file generated:

    ```sh
    cat build/ood-portal.conf
    ```
    
    If you made a mistake, clean up first then build again:
    
    ```sh
    scl enable rh-ruby22 -- rake clean
    scl enable rh-ruby22 -- rake ...
    ```

5.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => /opt/rh/httpd24/root/etc/httpd/conf.d/ood-portal.conf
    ```

6. If using default authentication setup (i.e., you didn't modify any of the environment variables attributed to authentication when building the config), be sure to create an `.htpasswd` file that **maps to system-level usernames**. The default location specified for this file is:
    
    ```
    /opt/rh/httpd24/root/etc/httpd/.htpasswd
    ```
        
    You can generate this file using the `htpasswd` binary as such:
        
    ```sh
    sudo scl enable httpd24 -- htpasswd -c /opt/rh/httpd24/root/etc/httpd/.htpasswd <username1>
    # => New password:
    ```
        
    The password doesn't need to correspond to the system-level password used by the user to log into the cluster. To add more accounts you can remove the `-c` option:
        
    ```sh
    sudo scl enable httpd24 -- htpasswd /opt/rh/httpd24/root/etc/httpd/.htpasswd <username2>
    # => New password:
    ```

Note: This package references the location of `mod_ood_proxy`, `nginx_stage`, and `ood_auth_map`. It is the source of knowledge for the locations of the various OOD infrastructure pieces. Be sure to update these locations if you change the `PREFIX` for any installation of the corresponding package.

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
    # As of writing this the latest version is `v0.0.6`
    git checkout tags/v0.0.6
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
    # As of writing this the latest version is `v0.1.0`
    git checkout tags/v0.1.0
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

## Section 3. App Deployment Strategy

This is the strategy currently employed at the OSC OnDemand and OSC AweSim portals for deploying apps. This is in no way a requirement, and system administrators are encouraged to explore different options.

### 3.1 - Local Directory Structure

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

### 3.2 - Mapping URI to Local Directory Structure

#### 3.2.1 - Apps URI

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

#### 3.2.2 - Public URI

**Anyone** can access the resources underneath the `/public` URI.

```sh
# Accessing a publicly available resource
https://<PORTAL>.osc.edu/public/favicon.ico  =>  /var/www/ood/public/favicon.ico
```

### 3.3 - OSC Portals

Available OSC Portals:

| `<PORTAL>` | `<HOST>` |
| ---------- | -------- |
| `ondemand` | `web02.hpc.osc.edu` |
| `awesim`   | `web04.hpc.osc.edu` |

#### 3.3.1 - Add/Update System Apps (requires root)

The system apps are maintained by mirroring the following directory:

```
/users/PZS0645/wiag/ood_portals/<PORTAL>/sys
```

The command to mirror this directory (**performed** on `webXX`) is:

```sh
# Mirror the deployment directory with the staged directory
sudo rsync -rlptvu --delete /users/PZS0645/wiag/ood_portals/<PORTAL>/sys/ /var/www/ood/apps/sys
```

#### 3.3.2 - Add/Update User Apps (requires root)

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

#### 3.3.3 - Add/Update Dev Apps

The user will need to create the required dev directory in their home directory in order to develop apps on the given `<PORTAL>`:

```sh
# Create directory that holds dev apps
mkdir -p ~/<PORTAL>/dev
```

#### 3.3.4 - Add/Update Public Assets (requires root)

The public assets are maintained by mirroring the following directory:

```
/users/PZS0645/wiag/ood_portals/<PORTAL>/public
```

The command to mirror this directory (**performed** on `webXX`) is:

```sh
# Mirror the deployment directory with the staged directory
sudo rsync -rlptvu --delete /users/PZS0645/wiag/ood_portals/<PORTAL>/public/ /var/www/ood/public
```

## Section 4. System Apps

These are the apps deployed by the system administrator on the local disk that are accessible by all users.

### 4.1 - Dashboard App

See https://github.com/OSC/ood-dashboard for more information.

### 4.2 - Shell App

See https://github.com/OSC/ood-shell for more information.

### 4.3 - Files App

See https://github.com/OSC/ood-fileexplorer for more information.

### 4.4 - File Editor App

See https://github.com/OSC/ood-fileeditor for more information.

### 4.5 - User Documentation

Currently there is no general user documentation provided.

OSC provides user documentation for their Open OnDemand deployment (OSC OnDemand) at:

https://www.osc.edu/resources/online_portals/ondemand

Feel free to use it as template for your own organization's user-facing documentation.

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


