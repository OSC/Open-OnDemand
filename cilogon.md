---
layout: default
title: CILogon Strategy
---

**Warning**: Below is not for the faint of heart. It is missing documentation
and can be very difficult to fully grasp. It is not recommended to deploy a
CILogon authentication strategy on your own. You have been warned!

The current strategy employed by the OSC OnDemand portal is outlined in detail
in the figure below.

[![Authentication Strategy](http://www.plantuml.com/plantuml/png/jLLHRzem47xFhxX7U82WxWrfGrMBgahRYe1f3wEgNFi8reBFiWrhzkjd1mXa6CZQzIbAVFTzT_Tzvp3koC1rBZaccXadkTHCEVYEprz9rB_Tt7-cZ5JjsxqO9KcP3frFkwm-t0sdi71EstwQBiYwb6cTCExacZcimBVFMYPfL59mi6Yj93LJ-2Aj6q1BKNkVu3DAqZxy-D2xUeGn2ccGkpgfa9RJ6aqYy8YxIT5L_kdA3mgvwlKUuNQRKi28PVXQOl8JI0VIse_TRhSH-mJTVdGk17EYdn9dwiI0kCo3c5qoMSfMRa56_iP1e3jRyzO3zPfFOrL8E-yeyYzXG7kabVLlV47a9Gz-0gR4JDaLvzu2yrMU5n10KNdJUIYRR3ERkc0ppBywszNJN8tpIXpaSuCXXEqjFAzMBABSBUOOPX8LhoRh1kwIiDMwlT9xRz_K0cqxcb4glVG4A0R777jhL0a-5iR3Bm_JuTEtoN0CZi0Mrk7oAK3DLCi_83DI-C3ENEROJaWxK5IoI9N1d5PA902nSXJvWyYK3gARESnWvZj7Cp3XkDpP4y37yxyCTAw-ndB4Uoglh759QfWBvYflhSR5ZnucqqiTGEkPHB4HNx4bfffP-ukCQBz3RuTcqo801tTql1k2EsPApc7DSYd2aefmex3S_hlUOKKQpoYwOE_R-U6QHPd-pwgXtwmOloyjTld4GhMXbo9mXQ6STAW_W6ljwqEhITdOR_KF)](http://www.plantuml.com/plantuml/uml/jLPDRnen4BtlhvWZ761HUaDgITIWAbBR8A3g8KIHOm-BrSLZsLwQxQ-llNnkCR1DczDgrpFptinxOxYsZXbNBFE4SKS6RdB9CT7-c-HIC-NWEpv_9L7_zl7kCsMaQj_kn2X9gdJeVDXczU7JS0eBvxJjf-c2h4kPrW5BhfZ3NV7DivP96fKK72nQAqdD5Bv8wmRGKjJU9_YC2hGF3-zwLuymHY4DSbS75T9oMg5911vXDo6zop_D_838LQ_t2BURb196BCFN0fEVG3gGrhxlThkDq2VezgDp8PWJ_P0uKoS6n6KUnEoIoL9Mk0KP-Gi70Uuivuq7wfnFrzfLsdf7aNy90zfNAgn_umiYBtdm4p0bPieMd7iBp8iyBo4KHULDvqlPP9FPr0NhOVxHsPPFSpNEAt6Gpms64BQty7otOX7bRZ77C9EeU3LPr-Kk2RjRjrTwxzytjKJRZcPKIYyz08h1CSRU6hi47mlZuPVxwV3fssGu1aTWI-jm-HHApLJBFo0pKlZ0pbpcs4v8Er1KeKYLmPmA9HA061r5_Y0ofOFePWupcFdEqGnCEAvszaJXu_bVBdIENsCvuZrLLzQu0ZNC1NDLjrPZygFxoVHI1z1w9b6knLTiYQbcrlua8zflqDkXsRG8XOCxEjuDmHqp9UUmOhaKOKb5k55ORlzUxx0YZMSKNR2txVnmpQ98_sVLq6zM3DzJYrq-iL1jwAKCM64e9ntg3-16UtveDIaR_Lczzcpy7vhV)

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
