---
layout: default
title: Installation
---

This installation tutorial starts with a web host `webdev05.osc.edu` which has

* the users are mirrored with the other machines (LDAP+NSS)
* the home directory shared file system is mounted and accessible by users
* TORQUE client libraries installed and the `trqauthd` process started
* signed SSL certificates with corresponding intermediate certificate
* Software Collections package/repo, `lsof`, and `sudo` are installed
* the host has been added as a submit host to the respective batch servers

## Software Requirements

We will use RedHat Software Collections to satisfy these requirements:

* Apache HTTP Server 2.4
* NGINX 1.6
* Phusion Passenger 4.0
* Ruby 2.2 with rake, bundler, and ability to build gem native extensions
* Node.js 0.10
* Git 1.9

In this tutorial scl (RedHat Software Collections) packages happen to be
installed at `/opt/rh`. This tutorial is also done from an account that has
sudo access but is not root.

Enable the RHSCL repository. Note that you may also need to enable the Optional
channel and attach a subscription providing access to RHSCL to be able to use
the repository.

```
sudo subscription-manager repos --enable=rhel-server-rhscl-6-rpms
# => Repository 'rhel-server-rhscl-6-rpms' is enabled for this system.
```

Install dependencies:

```
sudo yum install -y httpd24 nginx16 rh-passenger40 rh-ruby22 rh-ruby22-rubygem-rake rh-ruby22-rubygem-bundler rh-ruby22-ruby-devel nodejs010 git19
```

Update Apache Environment to include `rh-ruby22`. This is necessary for the user mapping
script written in Ruby. Do this by editing `/opt/rh/httpd24/service-environment`:

```diff
-HTTPD24_HTTPD_SCLS_ENABLED="httpd24"
+HTTPD24_HTTPD_SCLS_ENABLED="httpd24 rh-ruby22"
```

Finally, make a source directory that will contain the checked out and built
OOD infrastructure components and apps:

```sh
mkdir -p ~/tmp/ood/src
```

## Generate Apache Config

1.  Clone and check out the latest tag:

    ```sh
    cd ~/tmp/ood/src
    scl enable git19 -- git clone https://github.com/OSC/ood-portal-generator.git
    cd ood-portal-generator/
    scl enable git19 -- git checkout v0.3.1
    ```

2.  `ood-portal-generator` is a script that takes a `config.yml` (or uses
    defaults if not provided) and renders an Apache config from a template.
    Generate a default one now:

    ```sh
    scl enable rh-ruby22 -- rake
    # => mkdir -p build
    # => rendering templates/ood-portal.conf.erb => build/ood-portal.conf
    ```

3.  Copy this to the default installation location:

    ```
    sudo scl enable rh-ruby22 -- rake install
    # => cp build/ood-portal.conf /opt/rh/httpd24/root/etc/httpd/conf.d/ood-portal.conf
    ```

4.  For now, lets use basic auth with an `.htpasswd` file until we get the
    installation complete. Then we will add another authentication mechanism.
    Start by generating an `.htpasswd` file with a user that **exists** on your
    system (the password need not be the same as their current system password):

    ```sh
    sudo scl enable httpd24 -- htpasswd -c /opt/rh/httpd24/root/etc/httpd/.htpasswd efranz
    #=> New password:
    #=> Re-type new password:
    #=> Adding password for user efranz
    ```

_Note: The Apache config references the location of `mod_ood_proxy`,
`nginx_stage`, and `ood_auth_map`. Be sure to update these locations if you
change the `PREFIX` for any installation of the corresponding package in the
`config.yml` prior to generating the Apache config._

## Install Proxy Module for Apache

An Apache module written in Lua is the primary component for the proxy logic. It is given by the [mod_ood_proxy](https://github.com/OSC/mod_ood_proxy) project.

1.  Clone and check out the latest tag:

    ```sh
    cd ~/tmp/ood/src
    scl enable git19 -- git clone https://github.com/OSC/mod_ood_proxy.git
    cd mod_ood_proxy/
    scl enable git19 -- git checkout v0.2.0
    ```

2.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => mkdir -p /opt/ood/mod_ood_proxy
    # => ...
    ```

## Install the PUN Utility

The PUNs are manipulated and maintained by the
[nginx_stage](https://github.com/OSC/nginx_stage) utility. This tool is meant
to by run by `root` or a user with `sudoers` privileges.

1.  Clone and check out the latest tag:

    ```sh
    cd ~/tmp/ood/src
    scl enable git19 -- git clone https://github.com/OSC/nginx_stage.git
    cd nginx_stage/
    scl enable git19 -- git checkout v0.2.1
    ```

2.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => /opt/ood/nginx_stage
    ```

    This creates the `nginx_stage` config
    `/opt/ood/nginx_stage/config/nginx_stage.yml` and the ruby binstub/wrapper
    script `/opt/ood/nginx_stage/bin/ood_ruby`.

    * If you run an older Linux OS that creates user accounts starting at id
      500, then you will need to modify `nginx_stage.yml` - the configuration
      option `min_uid: 1000` accordingly.


3.  Give the `apache` user `sudo` privileges to run the `nginx_stage` command.
    To do this, generate a `sudoers_ood` file in `~/tmp/ood/src` directory:

    ```
    Defaults:apache     !requiretty, !authenticate
    apache ALL=(ALL) NOPASSWD: /opt/ood/nginx_stage/sbin/nginx_stage
    ```

    and then copy this to `/etc/sudoers.d/10_ood`:

    ```sh
    sudo cp ~/tmp/ood/src/sudoers_ood /etc/sudoers.d/10_ood
    sudo chmod 440 /etc/sudoers.d/10_ood
    ```

    Our `/etc/sudoers` file includes files in `/etc/sudoers.d`:

    ```sh
    sudo tail -n 2 /etc/sudoers
    ## Read drop-in files from /etc/sudoers.d (the # here does not mean a comment)
    #includedir /etc/sudoers.d
    ```

## Install User Mapping Script

You will need to map the Apache authenticated user to the local system user.
This is done with the simple tool:
[ood_auth_map](https://github.com/OSC/ood_auth_map).

1.  Clone and check out the latest tag:

    ```sh
    cd ~/tmp/ood/src
    scl enable git19 -- git clone https://github.com/OSC/ood_auth_map.git
    cd ood_auth_map/
    scl enable git19 -- git checkout v0.0.3
    ```

2.  Install it to its global location:

    ```sh
    sudo scl enable rh-ruby22 -- rake install
    # => mkdir -p /opt/ood/ood_auth_map/bin
    # => ...
    ```

The principle behind this script is that you call it with a URL encoded
`REMOTE_USER` user name as the only argument, and it will return the mapping to
the local system user name if it exists.

## Add Cluster Connection Config Files

**(Optional step)**

The Dashboard, File Explorer, and Shell Access can work without cluster
connection config files. These config files are required for:

* enable Shell Access to multiple named hosts outside of the local host OOD is
  running on
* use Active Jobs, My Jobs, or any other app that works with batch jobs


1. Create default directory for config files:

    ```sh
    sudo mkdir -p /etc/ood/config/clusters.d
    ```

2. Add one config file for each host you want to provide access to. Each config
   file is a YAML file and must have the `.yml` suffix.

Here is the minimal YAML config required to specify a host that can be accessed
via Shell Access app. The filename is `oakley.yml`:

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

* a cluster contains one ore more servers, with server names `login`,
  `resource_mgr` and `scheduler`
* special server keywords are:
  * `login`
  * `resource_mgr`
  * `scheduler`
  * `ganglia`

For Active Jobs and My Jobs to work, a cluster configuration must specify a
`resource_mgr` to use.

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

The name of the file becomes the key for this host. So `oakley.yml` cluster
config will have a key `oakley`. My Jobs and other OOD apps that cache
information about jobs they manage will associate job metadata with this key.

## Disable SELinux and open port 80 and 443 through IP Tables

1. Disable SELinux (not documented here)

2. Open port 80 and 443 through IP Tables i.e.

    ```sh
    sudo iptables -I INPUT -p tcp -m multiport --dports 80 -m comment --comment "00080 *:80" -j ACCEPT
    sudo iptables -I INPUT -p tcp -m multiport --dports 443 -m comment --comment "00443 *:443" -j ACCEPT
    sudo /etc/init.d/iptables save
    ```

## Start Apache

```sh
sudo service httpd24-httpd start
# => Starting httpd:                                            [  OK  ]
```

If you access the host now through a web browser you should see this error:

```
Error -- invalid app root: /var/www/ood/apps/sys/dashboard
Run 'nginx_stage --help' to see a full list of available command line options.
```

Success! The infrastructure components are installed and now we need to install
the OOD "System Apps".

## Add SSL Support

**(Optional step, but recommended)**

SSL provides for an encrypted channel of communication between the user's
browser and the Open OnDemand portal. This requires:

* a server name that points to the Open OnDemand server
  (`webdev05.hpc.osc.edu`)
* signed SSL certificates with possible intermediate certificates

In this example the certificates are located at:

```sh
# Public certificate
/etc/pki/tls/certs/webdev05.hpc.osc.edu.crt

# Private key
/etc/pki/tls/private/webdev05.hpc.osc.edu.key

# Intermediate certificate
/etc/pki/tls/certs/webdev05.hpc.osc.edu-interm.crt
```

1.  Install the necessary Apache module to use SSL:

    ```sh
    sudo yum install httpd24-mod_ssl.x86_64
    ```

2.  Update the Apache config with this server name and paths to the SSL
    certificates.

    ```sh
    cd ~/tmp/ood/src/ood-portal-generator
    ```

    Create or edit the `config.yml` as such:

    ```yaml
    ---

    servername: webdev05.hpc.osc.edu
    ssl:
      - 'SSLCertificateFile "/etc/pki/tls/certs/webdev05.hpc.osc.edu.crt"'
      - 'SSLCertificateKeyFile "/etc/pki/tls/private/webdev05.hpc.osc.edu.key"'
      - 'SSLCertificateChainFile "/etc/pki/tls/certs/webdev05.hpc.osc.edu-interm.crt"'
    ```

    For documentation on SSL directives please see
    https://httpd.apache.org/docs/2.4/mod/mod_ssl.html

    Re-build the Apache config:

    ```sh
    scl enable rh-ruby22 -- rake
    ```

    Copy it over to the default location:

    ```sh
    scl enable rh-ruby22 -- rake install
    ```

3.  Restart the Apache server:

    ```sh
    sudo service httpd24-httpd restart
    ```

When you visit the portal in your browser now it should redirect any http
traffic to the proper https protocol.

```
https://webdev05.hpc.osc.edu/
```

## Add LDAP Support

**(Optional step, but recommended)**

LDAP support allows for your users to log in using their local username and
password. It also removes the need for the sys admin to keep updating the
`.htpasswd` file. This requires:

* an LDAP server preferably with SSL support (`cts06.osc.edu:636`)

1.  Install the necessary Apache module to use LDAP:

    ```sh
    sudo yum install httpd24-mod_ldap.x86_64
    ```

2.  Update the Apache config with LDAP Basic Authentication support.

    ```sh
    cd ~/tmp/ood/src/ood-portal-generator
    ```

    Create or edit the `config.yml` as such:

    ```yaml
    ---

    auth:
      - 'AuthType Basic'
      - 'AuthName "private"'
      - 'AuthBasicProvider ldap'
      - 'AuthLDAPURL "ldaps://cts06.osc.edu:636/ou=People,ou=hpc,o=osc?uid" SSL'
      - 'AuthLDAPGroupAttribute memberUid'
      - 'AuthLDAPGroupAttributeIsDN off'
      - 'RequestHeader unset Authorization'
      - 'Require valid-user'
    ```

    For documentation on LDAP directives please see
    https://httpd.apache.org/docs/2.4/mod/mod_authnz_ldap.html

    Re-build the Apache config:

    ```sh
    scl enable rh-ruby22 -- rake
    ```

    Copy it over to the default location:

    ```sh
    scl enable rh-ruby22 -- rake install
    ```

3.  Restart the Apache server:

    ```sh
    sudo service httpd24-httpd restart
    ```

Close your browser so that you are properly logged out. Then open your browser
again and access the portal. You should now be able to authenticate with your
local username and password.
