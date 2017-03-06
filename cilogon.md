---
layout: default
title: CILogon Strategy
---

**Warning**: Below is not for the faint of heart. It is missing documentation
and can be very difficult to fully grasp. It is not recommended to deploy a
CILogon authentication strategy on your own. You have been warned!

The current strategy employed by the OSC OnDemand portal is outlined in detail
in the figure below.

[![Authentication Strategy](http://www.plantuml.com/plantuml/png/jLLHRzem47xFhxX7U82WxWrfGzMAfahTYe1f3qjLkVOHh0MVPHlMxDVF3ZI1OKhQjKzHuhllkxllEuTnHmOk2yanqSmuoQLcoi4FVFX6ul3Rv-iRoaabIHKElKzFKKFuCfx3qZbjXscw9EjIfdN2k9CRvh06spqFCacZaeA3HMibgKpXexHk11r5tIR1PrIaG_ZvON1n1rCKqY1tXwH2MauRD8d08-usHTVvfolVA-HYCBY3grrA2HEMusk9oKyW7KbhF_Rx_LRiCtJDw5m8vaI_96Rgn82uB89uVJ9vojPkIKR-mL6WUxRcdUQ7DP_6gf6UlAB8luG1DKXhwz_uaiXB3jTY0ao9JDarvzu2YtLHb110KNdZUIYRx3BRkc0xp7ywUubt6u1M3kO6GmXhBBoeLogYt1KM66OI5Uz0rYtSEM7hTNkbz_v_KqkqtMX6fVJA4w0O7NCqrgWJV2nCnrzlv-FxxxFn51o1BQt3rNs0obJ7Fo0tKlZ0ta8Ms4r8Er1KiKYLmuBMIYG0iN8K-OF8bFQYspdCOEuxHpCouRZQsHF0n_C_3_HSVQtbYFTKNLdJYjGm5ynLtLgjYv_SpkRdEe3E0uboAxvYAqmri_Ot6T5zXzxspQP500xkw7at17TaIYvWmmefmfAASAEmmhvSxx0YhMSKNR3txNn_pQ9CRxDgw3ShnkzBYrq-iL1jwB4CN64e9ntg3-1I-_hGQb8szblz2m00)](http://www.plantuml.com/plantuml/uml/jLPHRzem47xFhxX7U82WxWrfezMAfahRYe1f3wEgNFi8reBFiXtMxDVF3ZI1OKhQhKzHuhll--xkEyEbTSouOfqdZ3ioS9LBZedstoINcYa7t_7XAud3RnzlFbD6AacgSEXzU8eQmgVn75REQJjCqnLPbpAj0xRSC8SrsEvva4aQbHGSB5ehIIqKFahhHj1Hr6qIV4P5EeGVxmONTp158GroTqWgfEMqGPC8FE9k8xhEFwryJyZ5O707rxkK4YOimzSIa-z0EfBMlk_t-wtOP-YQrhaGp8b-Iboe4mtYCW_3wvF9KbQu5Hdv6qU1xbdDkvaVQZwDLICzUKMHVmaDQf0Awz_uYiYF76x419WI2sKBphq5baMUbn10KNdZUIYRx3BRkc3RpBygUubt6u1M3kO6GmXhBBoiLoeokMlCCVGaEho3iMxXpWssNhTNUk_VDxL4surcL9DUUm8KesEEnXfrWe-5wVZBxNny_sqsde8ZiALrkBuFK0lLyG_8Z1G-iBUKENOJqWuKLInIPR2aGea482P7aJ-8T6alkft336Q-SZGZ4uwhdJq1VDp_yq3FwMizZdXDrPKrhKWDSy5SrQtLMlpaTZO_Lm5q7aYKs-4hjiPKCsj_aH7TDwXjtsnQ1E3WZfvtGt37GiabR5WkHLXI4MuKbjkNppkiIEiUHHViRIlVNxCeq_zPDVIR5UFtqcANJonK6tei0rTeoac7-WEuqbuVMatAHdzMBxqQ_mVc3m00)

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
