
## cpp.al notes  

[Ansible installation](#ansible-installation)  
[List Setup](#list-setup)  

## Ansible Installation

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

# Manually set this in /etc/postfix/main.cf. Ansible is not doing that.
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

