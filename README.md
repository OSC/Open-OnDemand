# Open-OnDemand

The Open OnDemand Project is an open-source software project, based on the Ohio Supercomputer Center's proven "OSC OnDemand" platform, that enables HPC centers to install and deploy advanced web and graphical interfaces for their users. More information can be found in the paper [http://dx.doi.org/10.1145/2949550.2949644](http://dx.doi.org/10.1145/2949550.2949644).

* [Components](#components)
  * [Proxy and PUN](#proxy-and-pun)
  * [Authentication and Authorization](#authentication-and-authorization)
* [Installation Guide](#installation-guide)
  * [Generate Apache Config for Open OnDemand Portal](#generate-apache-config-for-open-ondemand-portal)
  * [Install Open OnDemand Proxy Module for Apache](#install-open-ondemand-proxy-module-for-apache)
  * [Install the PUN Utility](#install-the-pun-utility)
  * [[Authentication] Install User Mapping Script](#authentication-install-user-mapping-script)
  * [[Authentication] Deploy the Discovery Page](#authentication-deploy-the-discovery-page)
  * [[Authentication] Deploy the Registration Page](#authentication-deploy-the-registration-page)
  * [[Authentication] Deploy Mapping Helper Scripts](https://github.com/OSC/Open-OnDemand/wiki#authentication-deploy-mapping-helper-scripts)
* [App Deployment Strategy](#app-deployment-strategy)
  * [Local Directory Structure](#local-directory-structure)
  * [Mapping URI to Local Directory Structure](#mapping-uri-to-local-directory-structure)
    * [Apps URI](#apps-uri)
    * [Discover URI](#discover-uri)
    * [Public URI](#public-uri)
    * [Register URI](#register-uri)
  * [OSC Portals](#osc-portals)
    * [Add/Update System Apps (requires root)](#addupdate-system-apps-requires-root)
    * [Add/Update User Apps (requires root)](#addupdate-user-apps-requires-root)
    * [Add/Update Dev Apps](#addupdate-dev-apps)
    * [Add/Update Public Assets (requires root)](#addupdate-public-assets-requires-root)
* [System Apps](#system-apps)
    * [Dashboard App](#dashboard-app)
    * [Shell App](#shell-app)
    * [Files App](#files-app)

## Components

Details of the components that make up the Open OnDemand **infrastructure**.

### Proxy and PUN

The core of the infrastructure includes a proxy layer that all traffic passes through using the securely encrypted SSL protocol on port 443. The [Apache proxy](https://httpd.apache.org/) parses the URI and dynamically determines where to route the traffic to. In most cases the traffic will be routed to the per-user [NGINX](https://www.nginx.com/) (PUN) web server.

The PUN is described as an NGINX server instance running as a system-level user listening on a [Unix domain socket](https://en.wikipedia.org/wiki/Unix_domain_socket). File and directory permissions are used to restrict access to this Unix domain socket such that only the proxy daemon can communicate with the PUN.

| Component | Description |
| --------- | ----------- |
| [ood-portal-generator](https://code.osu.edu/open-ondemand/ood-portal-generator) | Generates an Open OnDemand portal config for an Apache server that defines the proxy interface. |
| [mod_ood_proxy](https://code.osu.edu/open-ondemand/mod_ood_proxy) | An Apache httpd module implementing the Open OnDemand proxy API. |
| [nginx_stage](https://code.osu.edu/open-ondemand/nginx_stage) | Stages and controls the per-user NGINX (PUN) instances. |

### Authentication and Authorization

There is **no required** authentication mechanism built-into Open OnDemand, but we do provide a recommended solution. The recommended solution utilizes the [mod_auth_openidc](https://github.com/pingidentity/mod_auth_openidc) module within the Apache proxy to authenticate users against an [OpenID Connect](http://openid.net/connect/) Provider for federated authentication.

After the user authenticates with their OpenID Connect Provider authorization is granted by mapping the user's authenticated username to a local system username. The Apache proxy handles this by calling a script that handles the user mapping lookup. If the mapping fails, the user can be taken to a registration page where he or she can set up a mapping.

| Component | Description |
| --------- | ----------- |
| [mapdn](https://code.osu.edu/open-ondemand/mapdn) | Scripts to setup/maintain mappings between Distinguished Names (DNs) to local usernames. |
| [ood_auth_map](https://code.osu.edu/open-ondemand/ood_auth_map) | The user mapping script employed by OSC for OnDemand and AweSim. |
| [ood_auth_discovery](https://code.osu.edu/open-ondemand/ood_auth_discovery) | Open ID Connect Discovery page for OSC OnDemand. |
| [ood_auth_registration](https://code.osu.edu/open-ondemand/ood_auth_registration) | OSC OnDemand Open ID Connect registration page. |

## Installation Guide

**Work in Progress**

**Assumes you are using Software Collections**

### Generate Apache Config for Open OnDemand Portal

In this section we will generate an Open OnDemand Portal config file used by the Apache server. This can be done manually or using the [ood-portal-generator](https://code.osu.edu/open-ondemand/ood-portal-generator).

1.  We clone the `ood-portal-generator` repo to the local disk:

    ```sh
    git clone git@code.osu.edu:open-ondemand/ood-portal-generator.git
    cd ood-portal-generator
    ```

2.  Check out the version of the generator you intend on using:

    ```sh
    # A list of versions & details can be viewed in the CHANGELOG.md
    # As of writing this the latest version is `v0.0.1`
    git checkout tags/v0.0.1
    ```

3.  Now we build the Apache config using environment variables to modify any of the default settings:

    ```sh
    scl enable rh-ruby22 -- rake
    ```

    Documentation on the available options can be found at:

    https://code.osu.edu/open-ondemand/ood-portal-generator#default-options

    **SUBDOMAIN** -- If a subdomain or different IP address is used (assuming the SSL certificates are setup as well for this subdomain):

    ```sh
    scl enable rh-ruby22 -- rake OOD_IP=<ip> OOD_SERVER_NAME=<subdomain>

    # Example:
    scl enable rh-ruby22 -- rake OOD_IP='xxx.xxx.xxx.xxx' OOD_SERVER_NAME='ondemand.osc.edu'
    ```

4.  Confirm the Apache config is to your liking by viewing the config file generated:

    ```sh
    cat build/ood-portal.conf
    ```

5.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => /opt/rh/httpd24/root/etc/httpd/conf.d/ood-portal.conf
    ```

6.  If using recommended OOD authentication, be sure to setup the OpenID Connect Discovery Provider information needed by the `mod_auth_openidc` Apache module:

    https://code.osu.edu/open-ondemand/ood-portal-generator#cilogon-setup

    For CILogon you will need to register a **new** redirect URI for every host machine you install an OOD Portal on.

Note: This package references the location of `mod_ood_proxy`, `nginx_stage`, and `osc-user-map`. It is the source of knowledge for the locations of the various OOD infrastructure pieces. Be sure to update these locations if you change the `PREFIX` for any installation of the corresponding package.

### Install Open OnDemand Proxy Module for Apache

An Apache module written in Lua is the primary component for the proxy logic. It is given by the [mod_ood_proxy](https://code.osu.edu/open-ondemand/mod_ood_proxy) project.

1.  We clone the `mod_ood_proxy` repo to the local disk:

    ```sh
    git clone git@code.osu.edu:open-ondemand/mod_ood_proxy.git
    cd mod_ood_proxy
    ```

2.  Check out the version of the generator you intend on using:

    ```sh
    # A list of versions & details can be viewed in the CHANGELOG.md
    # As of writing this the latest version is `v0.0.1`
    git checkout tags/v0.0.1
    ```

3.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => /opt/ood/mod_ood_proxy
    ```

### Install the PUN Utility

The PUNs are manipulated and maintained by the [nginx_stage](https://code.osu.edu/open-ondemand/nginx_stage) utility. This tool is meant to by run by `root` or a user with `sudoers` privileges.

1.  We clone the `nginx_stage` repo to the local disk:

    ```sh
    git clone git@code.osu.edu:open-ondemand/nginx_stage.git
    cd nginx_stage
    ```

2.  Check out the version of the generator you intend on using:

    ```sh
    # A list of versions & details can be viewed in the CHANGELOG.md
    # As of writing this the latest version is `v0.0.7`
    git checkout tags/v0.0.7
    ```

3.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => /opt/ood/nginx_stage
    ```

4.  Modify `/opt/ood/nginx_stage/config/nginx_stage.yml` and `/opt/ood/nginx_stage/bin/ood_ruby`.

    In particular, opt into metrics reporting in `nginx_stage.yml` and use Software Collections in `ood_ruby`.

    Note: These files will not be overwritten the next time you update `nginx_stage`.

5.  If not already done, give the `apache` user `sudo` privileges to run this tool:

    ```sh
    # /etc/sudoers

    Defaults:apache     !requiretty, !authenticate
    apache ALL=(ALL) NOPASSWD: /opt/ood/nginx_stage/sbin/nginx_stage
    ```

### [Authentication] Install User Mapping Script

If you are using the OOD recommended authentication procedure you will need to map the Apache authenticated user to the local system user. This is done with the simple tool: [ood_auth_map](https://code.osu.edu/open-ondemand/ood_auth_map).

1.  We clone the `ood_auth_map` repo to the local disk:

    ```sh
    git clone git@code.osu.edu:open-ondemand/ood_auth_map.git
    cd ood_auth_map
    ```

2.  Check out the version of the generator you intend on using:

    ```sh
    # A list of versions & details can be viewed in the CHANGELOG.md
    # As of writing this the latest version is `v0.0.1`
    git checkout tags/v0.0.1
    ```

3.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => /opt/ood/ood_auth_map
    ```

The principle behind this script is that you call it with a URL encoded `REMOTE_USER` user name as the only argument, and it will return the mapping to the local system user name if it exists. This requires the availability of the `/etc/grid-security/grid-mapfile` ([more information](http://toolkit.globus.org/toolkit/docs/2.4/gsi/grid-mapfile_v11.html)).

### [Authentication] Deploy the Discovery Page

Before a user is authenticated, the user is presented with a discovery page where he/she can choose the OpenID Connect Provider. For the [ood_auth_discovery](https://code.osu.edu/open-ondemand/ood_auth_discovery) repo it is a branded page with a link to CILogon.

1.  We clone the `ood_auth_discovery` repo to the local disk:

    ```sh
    git clone git@code.osu.edu:open-ondemand/ood_auth_discovery.git
    ```

2.  Install it to its global location:

    ```sh
    # create the /var/www/ood directory if it doesn't already exist

    # copy this repo to its default location
    sudo cp -r ood_auth_discovery /var/www/ood/discover
    ```

### [Authentication] Deploy the Registration Page

After a user is authenticated and it is determined that no mapping exists to a local system user, they are redirected to the [ood_auth_registration](https://code.osu.edu/open-ondemand/ood_auth_registration) branded page. Here the user is required to enter their local system credentials and the mapping is generated.

1.  We clone the `ood_auth_registration` repo to the local disk:

    ```sh
    git clone git@code.osu.edu:open-ondemand/ood_auth_registration.git
    ```

2.  Install it to its global location:

    ```sh
    # create the /var/www/ood directory if it doesn't already exist

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

### [Authentication] Deploy Mapping Helper Scripts

**FIXME**

## App Deployment Strategy

This is the strategy currently employed at the OSC OnDemand and OSC AweSim portals for deploying apps. This is in no way a requirement, and system administrators are encouraged to explore different options.

### Local Directory Structure

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

### Mapping URI to Local Directory Structure

#### Apps URI

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

#### Discover URI

**Anyone** can access the resources underneath the `/discover` URI.

```sh
# Accessing the discovery page
https://<PORTAL>.osc.edu/discover  =>  /var/www/ood/discover/index.php
```

#### Public URI

**Anyone** can access the resources underneath the `/public` URI.

```sh
# Accessing a publicly available resource
https://<PORTAL>.osc.edu/public/favicon.ico  =>  /var/www/ood/public/favicon.ico
```

#### Register URI

To access any resource underneath the `/register` URI you **MUST**:

  - be authenticated through Apache (in particular authenticated through CILogon)

```sh
# Accessing the registration page
https://<PORTAL>.osc.edu/register  =>  /var/www/ood/register/index.php
```

### OSC Portals

Available OSC Portals:

| `<PORTAL>` | `<HOST>` |
| ---------- | -------- |
| `ondemand` | `websvcs08` |
| `awesim`   | `websvcs??` |

#### Add/Update System Apps (requires root)

The system apps are maintained by mirroring the following directory:

```
/nfs/gpfs/PZS0645/ood_portals/<PORTAL>/sys
```

The command to mirror this directory (**performed** on `websvcsXX`) is:

```sh
# Mirror the deployment directory with the staged directory
sudo rsync -rlptvu --delete /nfs/gpfs/PZS0645/ood_portals/<PORTAL>/sys/ /var/www/ood/apps/sys
```

#### Add/Update User Apps (requires root)

Each `<USER>` directory on the local disk will be given an appropriate permission such that only members of the `<GROUP>` can access the user's shared apps. This is an important security condition so that users don't access malicious developers' apps.

| `<PORTAL>` | `<GROUP>` |
| ---------- | --------- |
| `ondemand` | `<USER>`'s primary group |
| `awesim`   | `awesim`  |

To register a developer with username `<USER>` and groupname `<GROUP>` (**performed** on `websvcsXX`):

```sh
# Create the user's app sharing directory
sudo mkdir -p /var/www/ood/apps/usr/<USER>

# Set proper permissions so only primary group members can access shared apps
# NB: This is important or any user can access this developer's apps which can be a security risk
sudo chmod 750 /var/www/ood/apps/usr/<USER>
sudo chgrp <GROUP> /var/www/ood/apps/usr/<USER>

# Create symlink to user's shared apps location in their homedir for given portal
# NB: May need to create the directory for the user in their home directory
sudo -u <USER> mkdir -p ~<USER>/<PORTAL>/share
sudo ln -s ~<USER>/<PORTAL>/share /var/www/ood/apps/usr/<USER>/gateway
```

Now the user can update/maintain his or her own apps within their home directory:

```sh
~/<PORTAL>/share/*
```

#### Add/Update Dev Apps

The user will need to create the required dev directory in their home directory in order to develop apps on the given `<PORTAL>`:

```sh
# Create directory that holds dev apps
mkdir -p ~/<PORTAL>/dev
```

#### Add/Update Public Assets (requires root)

The public assets are maintained by mirroring the following directory:

```
/nfs/gpfs/PZS0645/ood_portals/<PORTAL>/public
```

The command to mirror this directory (**performed** on `websvcsXX`) is:

```sh
# Mirror the deployment directory with the staged directory
sudo rsync -rlptvu --delete /nfs/gpfs/PZS0645/ood_portals/<PORTAL>/public/ /var/www/ood/public
```

## System Apps

These are the apps deployed by the system administrator on the local disk that are accessible by all users.

### Dashboard App

### Shell App

### Files App
