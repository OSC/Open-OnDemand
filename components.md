---
layout: default
title: Components
---

The components that make up the Open OnDemand **infrastructure** includes a
proxy layer that all traffic passes through using the securely encrypted SSL
protocol on port 443. The [Apache proxy](https://httpd.apache.org/) parses the
URI and dynamically determines where to route the traffic to. In most cases the
traffic will be routed to the per-user [NGINX](https://www.nginx.com/) (PUN)
web server.

The PUN is described as an NGINX server instance running as a system-level user
listening on a [Unix domain
socket](https://en.wikipedia.org/wiki/Unix_domain_socket). File and directory
permissions are used to restrict access to this Unix domain socket such that
only the proxy daemon can communicate with the PUN.

| Component                                                           | Release                                                                               | Description                                                                                     |
| ---------                                                           | -------                                                                               | -----------                                                                                     |
| [ood-portal-generator](https://github.com/OSC/ood-portal-generator) | ![GitHub Release](https://img.shields.io/github/release/osc/ood-portal-generator.svg) | Generates an Open OnDemand portal config for an Apache server that defines the proxy interface. |
| [mod_ood_proxy](https://github.com/OSC/mod_ood_proxy)               | ![GitHub Release](https://img.shields.io/github/release/osc/mod_ood_proxy.svg)        | An Apache httpd module implementing the Open OnDemand proxy API.                                |
| [ood_auth_map](https://github.com/OSC/ood_auth_map)                 | ![GitHub Release](https://img.shields.io/github/release/osc/ood_auth_map.svg)         | The user mapping script that maps the authenticated username to the system-level username.      |
| [nginx_stage](https://github.com/OSC/nginx_stage)                   | ![GitHub Release](https://img.shields.io/github/release/osc/nginx_stage.svg)          | Stages and controls the per-user NGINX (PUN) instances.                                         |
