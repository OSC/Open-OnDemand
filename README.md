# Open OnDemand

[![DOI](http://joss.theoj.org/papers/10.21105/joss.00622/status.svg)](https://doi.org/10.21105/joss.00622)


The Open OnDemand Project is an open-source software project, based on the Ohio
Supercomputer Center's proven "OSC OnDemand" platform, that enables HPC centers
to install and deploy advanced web and graphical interfaces for their users.
More information can be found in the paper
[http://dx.doi.org/10.1145/2949550.2949644](http://dx.doi.org/10.1145/2949550.2949644).

1. The website https://osc.github.io/Open-OnDemand/ provides links to past webinar recordings and other information.
2. The [documentation](https://osc.github.io/ood-documentation/master/) provides installation directions, app development tutorials, and an overview of the components and applications that make up OnDemand.

Don't hesistate to reach out to the developers via our [Discourse instance](https://discourse.osc.edu/c/open-ondemand) or the [mailing list](https://lists.osu.edu/mailman/listinfo/ood-users) if you would like more information or need help installing or configuring Open OnDemand.

## Main OnDemand repo

All of the components and core apps, excluding the appkit gems and generic interactive apps like Jupyter, are now part of a single repo: https://github.com/OSC/ondemand

## Appkit

| Name | GitHub URL |
| --- | --- |
| ood_core	| https://github.com/OSC/ood_core |
| ood_appkit	| https://github.com/OSC/ood_appkit |
| ood_support	| https://github.com/OSC/ood_support |
| osc-machete	| https://github.com/OSC/osc-machete (deprecated)|
| osc_machete_rails	| https://github.com/OSC/osc_machete_rails (deprecated) |

## Applications: Generic Interactive Apps

| Name | GitHub URL |
| --- | --- |
| Jupyter	| https://github.com/OSC/bc_example_jupyter |


## Applications: OSC's Interactive Apps

| Name | GitHub URL |
| --- | --- |
| ANSYS Workbench	| https://github.com/OSC/bc_osc_ansys_workbench |
| Abaqus/CAE	| https://github.com/OSC/bc_osc_abaqus |
| COMSOL Multiphysics	| https://github.com/OSC/bc_osc_comsol |
| MATLAB	| https://github.com/OSC/bc_osc_matlab |
| Jupyter	| https://github.com/OSC/bc_osc_jupyter |
| Jupyter + Spark	| https://github.com/OSC/bc_osc_jupyter_spark |
| Paraview	| https://github.com/OSC/bc_osc_paraview |
| RStudio Server	| https://github.com/OSC/bc_osc_rstudio_server |
| VMD	| https://github.com/OSC/bc_osc_vmd |


## Packaging

| Name | Description | GitHub URL |
| --- | --- | --- |
| ondemand-packaging | RPM spec files for OnDemand and dependencies | https://github.com/OSC/ondemand-packaging |
| ondemand | specifies component versions to include in OnDemand RPM | https://github.com/OSC/ondemand |
| ood-apps-installer | deprecated installer script (replaced with RPM) | https://github.com/OSC/ood-apps-installer |

## Documentation

| Name | GitHub URL |
| --- | --- |
| ood-documentation | https://github.com/OSC/ood-documentation |
| ood-documentation-test | https://github.com/OSC/ood-documentation-test |

## Legacy archived repos:

Components:

| Name | Legacy GitHub URL |
| --- | --- |
| ood-portal-generator	| https://github.com/OSC/ood-portal-generator |
| mod_ood_proxy	| https://github.com/OSC/mod_ood_proxy |
| ood_auth_map	| https://github.com/OSC/ood_auth_map |
| nginx_stage	| https://github.com/OSC/nginx_stage |

Apps:

| Name | GitHub URL |
| --- | --- |
| Dashboard	| https://github.com/OSC/ood-dashboard |
| Shell	| https://github.com/OSC/ood-shell |
| Files	| https://github.com/OSC/ood-fileexplorer |
| Editor	| https://github.com/OSC/ood-fileeditor |
| Active Jobs	| https://github.com/OSC/ood-activejobs |
| Job Composer	| https://github.com/OSC/ood-myjobs |
| Interactive Apps: Desktop	| https://github.com/OSC/bc_desktop |



## License

* Documentation, website content, and logo is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/)
* Code is licensed under MIT (see LICENSE.txt)
