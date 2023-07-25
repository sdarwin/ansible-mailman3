
## cpp.al notes  

[Overview](#overview)  
[Web Integration](#web-integration)  
[Ansible installation](#ansible-installation)  
[List Setup](#list-setup)  

## Overview

This Ansible role, originally based on https://github.com/galaxyproject/ansible-mailman3, will install either mailman-core (branch 
https://github.com/cppalliance/ansible-mailman3/tree/cppal-core) or both mailman-web and mailman-core (branch https://github.com/cppalliance/ansible-mailman3/tree/cppal-full).

In production, the cppal-core branch installs only the core component. The mailman-web package will be included in the main Django web project and therefore Ansible will not install it.  

## Web Integration

When adding mailman-web into the boost website, which settings need to be added?  An example [django setting.py file has been included here.](settings.py.example1)  

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
					base_url: https://mailman-staging.boost.cpp.al/archives/
					api_key: XYZ...   (matches settings.py on the other end)

mailman-core -> postorius		---

mailman-core -> db			/etc/mailman3/mailman.cfg ->
						[database]
						class: mailman.database.postgresql.PostgreSQLDatabase
						url: postgresql://mailman3_core:__passwd__!@staging-db1.boost.cpp.al/mailman3_core

```

In settings.py, HAYSTACK_CONNECTIONS deals with full text indexing. By default this uses the whoosh backend which stores files in /var/lib/mailman3/web/fulltext_index or the equivalent. However with docker/kubernetes the indexes ought to be hosted remotely instead of in the local filesystem. Switch to the Elasticsearch backend. Prior to implementing Elasticsearch, Hyperkitty archives should still work but without the 'search' feature.  

Mailman-web recommended cron jobs to handle periodic tasks. See the example cron file in this directory [cron.d.example1](cron.d.example1). Those should be translated to celery jobs on the web side.  

## Ansible Installation

The required list of variables will change depending on a core-only installation or also installing mailman-web.  

The following is for a full install, using the cppal-full branch.  

- Configure /etc/ansible/host_vars/mailman-staging.boost.cpp.al/vars   

Fill in passwords and secrets as necessary.  

```
mailman3_django_config:
  secret_key: ___
  # Don't define hyperkitty_attachment_folder. If not set, it will store attachments in the db.
  # hyperkitty_attachment_folder: "{{ mailman3_web_var_dir }}/attachments"
  csrf_trusted_origins:
    - http://mailman-staging.boost.cpp.al
    - https://mailman-staging.boost.cpp.al
  databases:
    default:
      ENGINE: 'django.db.backends.postgresql_psycopg2'
      NAME: 'mailman3_web'
      HOST: 'staging-db1.boost.cpp.al'
      USER: 'mailman3_web'
      PASSWORD: ___
      PORT: '5432'

mailman3_extra_packages:
  - psycopg2-binary

mailman3_config:
  mailman:
    site_owner: ___
  database:
    class: mailman.database.postgresql.PostgreSQLDatabase
    url: postgresql://mailman3_core:___@staging-db1.boost.cpp.al/mailman3_core

mailman3_django_superusers:
  - name: ___
    email: ___
    pass: ___

mailman3_archiver_key: ___
mailman3_hyperkitty_server_url: 'https://mailman-staging.boost.cpp.al'

# Manually set this in /etc/postfix/main.cf. Now that ansible is installing main.cf, it should already be present.
# transport_maps = hash:/var/lib/mailman3/data/postfix_lmtp
# local_recipient_maps = hash:/var/lib/mailman3/data/postfix_lmtp
# relay_domains = hash:/var/lib/mailman3/data/postfix_domains

mailman3_nginx_ssl_certs: |
  ssl_certificate /etc/letsencrypt/live/{{ mailman3_web_url }}/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/{{ mailman3_web_url }}/privkey.pem;
```

## List Setup  

After creating a list in the UI, make the following updates to the list:  

Settings -> DMARC mitigations -> DMARC mitigation action -> Replace From with list address. Save changes.  

Settings -> Alter messages -> Reply goes to list -> Reply goes to list  

Settings -> Alter messages -> Filter Content -> Yes  

Settings -> Alter messages -> Convert html to plaintext -> Yes. Save changes.   

