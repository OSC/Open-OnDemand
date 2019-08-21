# Security Policy

This document outlines security procedures and general policies for the `OnDemand`
project.

## Security Audits

We have used [Trusted CI](https://trustedci.org/) in the past, the NSF Cybersecurity Center of 
Excellence conducted an in-depth vulnerability assessment of Open OnDemand, completing 
it in December 2018. This assessment included a careful review of the code, increasing 
our confidence in its security. The Ohio Supercomputing Center addressed the implementation 
issues (bugs) that were found during this review, producing a more robust revision of Open OnDemand.

We hope to have more audits on a regular ongoing basis and to share those results when applicable.

## Reporting a Vulnerability


If you have security concerns or think you have found a vulnerability in Open OnDemand,
please contact us directly via [email](mailto:ood-users@lists.osc.edu) on the news list found [here](https://lists.osu.edu/mailman/listinfo/ood-users).

Your email will be answered promptly by one of our team.

## Disclosure Policy

When the team receives a security vulnerability, they will generally assign it 
to a primary handler. This person will coordinate the fix and release process,
involving the following steps:

  * Confirm the problem and determine the affected versions.
  * Audit code to find any potential similar problems.
  * Prepare fixes for all releases still under maintenance. These fixes will be
    released as fast as possible.

## Comments on this Policy

If you have suggestions on how this process could be improved please submit 
a ticket, open a [discorse](https://discourse.osc.edu/) topic or open a pull request.
