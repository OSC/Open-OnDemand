---
layout: default
title: App Deployment
---

Apps are deployed on the local disk of the host serving the Open OnDemand
portal. Care must be taken for user shared apps as they are deployed through
symlinks to the corresponding user's home directory on the shared file system.

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

  - be authenticated through Apache (in particular authenticated through
    CILogon)
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

| `<PORTAL>` | `<HOST>`            |
| ---------- | --------            |
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

Each `<USER>` directory on the local disk will be given an appropriate
permission such that only members of the `<GROUP>` can access the user's shared
apps. This is an important security condition so that users don't access
malicious developers' apps.

| `<PORTAL>` | `<GROUP>`                |
| ---------- | ---------                |
| `ondemand` | `<USER>`'s primary group |
| `awesim`   | `awesim`                 |

To register a developer with username `<USER>` and groupname `<GROUP>`
(**performed** on `webXX`):

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

Now the user can update/maintain his or her own apps within their home
directory:

```sh
~/<PORTAL>/share/*
```

#### 4.3.3 - Add/Update Dev Apps

The user will need to create the required dev directory in their home directory
in order to develop apps on the given `<PORTAL>`:

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
