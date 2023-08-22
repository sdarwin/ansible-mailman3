
## Core-only Installation 

[Overview](#overview)  
[Web Integration](#web-integration)  
[Ansible installation](#ansible-installation)  
[List Setup](#list-setup)  

## Overview

This Ansible role most often installs both mailman-web and mailman-core. However in the [mailman3-core branch](https://github.com/cppalliance/ansible-mailman3/tree/mailman3-core) of the git repository is a version of the role that only installs the core component and not the web.

If you embed mailman3-web into a larger Django web project, the other project will managed it. Perhaps deployment will be via Kubernetes or Docker. Then Ansible will only need to be concerned with mailman-core.

## Web Integration

When adding mailman-web into a website, which settings need to be added?  Review [an example django setting.py file.](settings.py.example1)  

Although those settings shouldn't be copy-pasted verbatim, most of them will be needed in some form or another.  Migrate those settings into the web project.  

Consider there are three applications (postorius, hyperkitty, mailman-core) all connecting between each other and the database, resulting in 9 possible connection paths. The filenames may change depending upon the deployment.   

```
postorius -> mailman-core		/var/lib/mailman3/web/project/settings.py ->
						# Mailman API credentials
						MAILMAN_REST_API_URL = 'http://localhost:8001'
						MAILMAN_REST_API_USER = 'restadmin'
						MAILMAN_REST_API_PASS = 'restpass'
						(matches /etc/mailman3/mailman.cfg)

postorius -> hyperkitty			---

postorius -> db				/var/lib/mailman3/web/project/settings.py -> DATABASES = { 'default': 

hyperkitty -> mailman-core		---

hyperkitty -> postorius			---

hyperkitty -> db			/var/lib/mailman3/web/project/settings.py -> DATABASES = { 'default': 

mailman-core -> hyperkitty		/etc/mailman3/hyperkitty.cfg ->
					[general]
					base_url: https://mailman.example.org/archives/
					api_key: XYZ...   (matches settings.py on the other end)

mailman-core -> postorius		---

mailman-core -> db			/etc/mailman3/mailman.cfg ->
						[database]
						class: mailman.database.postgresql.PostgreSQLDatabase
						url: postgresql://mailman3_core:__passwd__!@mailman.example.org/mailman3_core

```

In settings.py, HAYSTACK_CONNECTIONS deals with full text indexing. By default this uses the whoosh backend which stores files in /var/lib/mailman3/web/fulltext_index or the equivalent. However with docker/kubernetes the indexes ought to be hosted remotely instead of in the local filesystem. Switch to the Elasticsearch backend. Prior to implementing Elasticsearch, Hyperkitty archives should still work but without the 'search' feature.  

Mailman-web recommended cron jobs to handle periodic tasks. See the example cron file in this directory [cron.d.example1](cron.d.example1). Those should be translated to celery jobs on the web side.  

## Ansible Installation

A core-only installation requires fewer variables. Configure /etc/ansible/host_vars/mailman.example.org/vars. Fill in passwords and secrets as necessary.  

```yaml
mailman3_core_api_hostname: "{{ inventory_hostname }}"
mailman3_core_api_admin_user: adminuser
mailman3_core_api_admin_pass: adminpass
mailman3_archiver_key: archiverkey
mailman3_hyperkitty_server_url: 'https://mailman.example.org'
mailman3_config:
  mailman:
    site_owner: listmaster@example.org
  database:
    class: mailman.database.postgresql.PostgreSQLDatabase
    url: postgresql://mailman3_core:mypassword@mailman.example.org/mailman3_core
mailman3_extra_packages:
  - psycopg2-binary
mailman3_sasl_passwd: smtp.mailgun.org postmaster@mailman.example.org:555555555
mailman3_relayhost: smtp.mailgun.org:587
```

/etc/ansible/host_vars/database.example.org/vars:  

```
mailman3_database_list:
  - name: mailman3_core
    username: mailman3_core
    password: mypassword
    subnet: "0.0.0.0/0"
```

## List Setup  

Recommended changes. After creating a list in the UI, make the following updates to the list:  

Settings -> DMARC mitigations -> DMARC mitigation action -> Replace From with list address. Save changes.  

Settings -> Alter messages -> Reply goes to list -> Reply goes to list  

Settings -> Alter messages -> Filter Content -> Yes  

Settings -> Alter messages -> Convert html to plaintext -> Yes. Save changes.   

