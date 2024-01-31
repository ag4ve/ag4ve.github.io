---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

[PDF Resume](/static/docs/SRW_Resume_20240130.pdf)
### Shawn Wilson

| :----------------------------------------------------- | -------------------------------------: |
| [srwilson@ioswitch.dev](mailto:srwilson@ioswitch.dev)  | **PO Box 10905**                       |
| _PGP Fingerprint:_                                     | **Oakland, CA 94610**                  |
| **BD8A 77EE 8991 9B51 5D73 E795 B8AB 96D5 2BF0 B6D4**  | [(202) **505-3363**](tel:+12025053363) |

##### __PROFESSIONAL SUMMARY__:
- Experience in:
  + Writing queries in SQL, XPath, and REST
  + Using text processors and DSLs like jq, yq, csv, tsv, xml, TeX
  + x509, ocsp/crl, fpki, and extended UIDs in general
  + Crypto hardware: HSMs, smartcards, pgpcard, FIDO, and HC Vault/KMS/secrets management systems
  + Splunk and Elastic+logstash+fluentbit/syslog
  + Chef, Ansible, AWS Cloudformation, and Terraform
- Deep knowledge of:
  + regex and globs
  + bash
  + Perl, Ruby, Python
  + Linux (including some kernel level module/udev topics)
  + git (refer to my blog post on it – see link below)

##### __ACTIVITIES__:
I’m working on a script to iterate over all containers in a repository and look for secrets in each checkpoint and writing an IOS
Scriptable app to search through CVEs (from voice commands). I’m an Extra class Amateur Radio operator. I publish a blog on
random technologies that interest me.

| :------------------------------------------------------------------------------- | :---------------------------------------------------------------------------- |
| [Blog](http://ioswitch.dev)                                                      | [tmux-start](https://github.com/ag4ve/misc-scripts/blob/master/tmux-start.sh) |
| [Github](https://github.com/ag4ve)                                               | [bashpack](https://github.com/ag4ve/misc-scripts/blob/master/bashpack.pl)     |
| [NF-Save](https://github.com/ag4ve/NF-Save)                                      | [bash libs](https://github.com/ag4ve/bash-libs)                               |
| [mon-hosts](https://github.com/ag4ve/misc-scripts/blob/master/mon-hosts-packed)  | [nft policy](https://github.com/ag4ve/nft-policy)                             |


##### CERTIFICATIONS
CompTIA A+, Security+ and Linux+ 	

#### PROFESSIONAL EXPERIENCE

##### DevOps Engineer
###### Jotform – San Francisco, CA _Nov_ _2022_ _–_ _Sep_ _2023_
 * Maintained servers and infrastructure on Google Cloud platform
 * Created new Terraform modules and Ansible roles
 * Wrote a Bash script to look at each cert running on a server and report on how many days until it expired

##### Junior Vulnerability Management Engineer
###### Jacobs – Herndon, VA  _Jan_ _2021_ _–_ _Aug_ _2021_
 * Analyzed remediation and false positive submissions for accuracy
 * Built a powershell script to flag false positives

##### Senior Systems Admin
###### Innotac (contract: USCIS) – Falls Church, VA _Feb_ _2016_ _–_ _July_ _2020_
 * Worked between 3 AWS accounts containing multiple VPCs each and deployments in 12-17 unique environments
 * Designed and implemented a deployment strategy for RHEL/CentOS systems in Amazon Web Services cloud (create a shell script, cloudformation template
 * Managed a Chef upgrade (version 11 to 14) and repo cleanup
 * Implemented Hashicorp Vault (including OIDC sign in and AWS instance authentication for host secrets)
 * Created a chef resource (LWRP) to create iptables rules from protocol/application rule definitions and created a Splunk dashboard to show iptables log data across all servers
 * Investigated and explained or remediated security audit findings
 * Made sure that deployments only used internal resources
 * Created Groovy libraries and workflows to allow push button deployments and environment updates in Jenkins
 * Created/maintained Chef and Jenkins integrations with each other and AWS (boto3), packer, vault, Chef Minimart, etc.

##### Systems Admin
###### KoreLogic Security – Deale, MD _Dec_ _2012_ _–_ _Dec_ _2015_
 * Maintained Gentoo and Ubuntu Linux systems at three different geographic locations
 * Maintained password cracking hardware, Raritan KVMs, and other hardware
 * Wrote a Perl script that analyzes and efficiently presents data from iptables log lines
 * Wrote a Perl module that generates iptables rules from a Perl data structure (NF-Save)
 * Wrote a Bash script that starts tmux sessions and runs predefined commands
 * Wrote a Perl script that packs sourced bash 'modules' sourced from a script
 * Wrote a bash script to monitor hosts (mon-hosts)

