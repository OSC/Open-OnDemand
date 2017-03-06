---
layout: default
title: CILogon Strategy
---

**Warning**: Below is not for the faint of heart. It is missing documentation
and can be very difficult to fully grasp. It is not recommended to deploy a
CILogon authentication strategy on your own. You have been warned!

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

## Discovery Page

Before a user is authenticated, the user is presented with a discovery page
where he/she can choose the OpenID Connect Provider. For the
[ood_auth_discovery](https://github.com/OSC/ood_auth_discovery) repo it is a
branded page with a link to CILogon.

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

## Registration Page

After a user is authenticated and it is determined that no mapping exists to a
local system user, they are redirected to the
[ood_auth_registration](https://github.com/OSC/ood_auth_registration) branded
page. Here the user is required to enter their local system credentials and the
mapping is generated.

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

3.  This repo makes use of a securely encrypted `ldaps://...` server for
    authenticating a user to the local system. This requires the appropriate
    OpenLDAP Certificate Authority files be hosted on the local machine:

    ```sh
    /etc/openldap/cacerts/*
    ```

4.  This repo also generates a proper user mapping after the user authenticates
    successfully on the local system. So the `apache` user will need
    permissions to generate this mapping on behalf of the user. This requires
    granting the `apache` user `sudo` privileges to the respective `mapdn`
    script.

    ```sh
    # /etc/sudoers

    apache      ALL=(ALL) NOPASSWD:  /usr/local/bin/add-user-dn.real, \
                                     /usr/local/bin/delete-user-dn.real, \
                                     /usr/local/bin/list-user-dns.real
    ```

To access any resource underneath the `/register` URI you **MUST**:

  - be authenticated through Apache (in particular authenticated through
    CILogon)

```sh
# Accessing the registration page
https://<PORTAL>.osc.edu/register  =>  /var/www/ood/register/index.php
```

## Mapping Helper Scripts

**FIXME**

[ood_auth_mapdn](https://github.com/OSC/ood_auth_mapdn)
