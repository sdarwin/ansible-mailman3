Mailman 3 Ansible Role
======================

`cppalliance.mailman3` is an Ansible role to install the [Mailman 3][mailman3] suite, based on [galaxyproject.mailman3](https://github.com/galaxyproject/ansible-mailman3).
While `galaxyproject.mailman3` installed all components of the suite: Mailman 3 Core, and the Django-based web interfaces, Postorius and Hyperkitty, this current role
also adds basic installations of PostgreSQL, Postfix, and Nginx. Those 3rd party components are required by mailman3. However the option remains to install and 
manage those separately if you prefer.  

Mailman 3 is a complicated service with many parts. It is a good idea to familiarize yourself with them before
proceeding. See the [architecture documentation][mailman3_arch] for details.

See [docs/mailman3-core.md](docs/mailman3-core.md) for a version of this role that only installs mailman3-core and _not_ the web component.  

[mailman3]: https://docs.mailman3.org/
[mailman3_arch]: https://docs.mailman3.org/en/latest/architecture.html

Requirements
------------

Test has mainly been done on Ubuntu 22.04. Pull requests are welcome to verify and re-enable other OS versions.  

Role Variables
--------------

Variables are described in detail in [`defaults/main.yml`](defaults/main.yml).

Dependencies
------------

A production Mailman 3 service typically needs Postfix, PostgreSQL, and Nginx. If you choose to install them separately, the following roles may be useful:

- [galaxyproject.postgresql](https://galaxy.ansible.com/galaxyproject/postgresql)
- [galaxyproject.postgresql_objects](https://galaxy.ansible.com/galaxyproject/postgresql_objects)
- [galaxyproject.postfix](https://galaxy.ansible.com/galaxyproject/postfix)
- [galaxyproject.opendkim](https://galaxy.ansible.com/galaxyproject/opendkim)
- [galaxyproject.nginx](https://galaxy.ansible.com/galaxyproject/nginx)

A default installation of this role `cppalliance.mailman3` will include those packages, as discussed below.   

PostgreSQL
----------

The playbook `playbooks/mailman3-playbook.yml` includes a database-specific play and will call `postgres.yml` to install the database.  

```
---
- hosts: mailman3-database
  tasks:
    - name: Install DB
      ansible.builtin.include_role:
        name: cppalliance.mailman3
        tasks_from: database/main.yml
```

Decide which server will host the database. It might be the mailman instance or a backend DB server. Add that server to the inventory (the hosts file) in the `mailman3-database` group. Run the playbook.

Otherwise, to manage the DB externally, don't put any hosts to the `mailman3-database` group. Or modify the play. Follow your own method to install a database.                         

Postgres configuration variables (named mailman3_database_) can be found in `defaults/main.yml`.

Postfix
-------

The factor determining whether Postfix will be installed is the `mailman3_mta` variable in defaults/main.yml. If you choose to set the value of `mailman3_mta` to anything other than `postfix` it will avoid a Postfix installation.

Postfix related variables include `mailman3_relayhost` and `mailman3_sasl_passwd`. You _should_ configure those, but you may also choose to set each of those variables to an empty string `''` so they are not added in postfix/main.cf.  

`postfix/main.cf` is a role template and may be customized or replaced.  

Nginx
-----

The factor determining whether Nginx will be installed is the `mailman3_install_nginx` variable in defaults/main.yml. If you choose to set the value to `false` it will avoid an Nginx installation.

This role doesn't install SSL certificates, but you should likely add them. Review the `mailman3_nginx_ssl_certs` variable which starts out using self-signed certificates. After you have installed valid certificates set that variable to use another configuration.  

Example Playbook
----------------

A playbook has been included in playbooks/mailman3-playbook.yml:  

```
---
- hosts: mailman3-database
  become: true
  remote_user: ubuntu
  tasks:
    - name: Install DB
      ansible.builtin.include_role:
        name: cppalliance.mailman3
        tasks_from: database/main.yml

- hosts: mailman3
  become: true
  remote_user: ubuntu
  roles:
    - role: cppalliance.mailman3

```

In the inventory `hosts` file add servers to the `mailman3` and `mailman3-database` groups. Configure variables in a file such as host_vars/mailman.example.org/vars.  
The following example uses a network connection to reach the database:  

```yaml
mailman3_django_superusers:
  - name: admin
    pass: admin
    email: admin@example.org
mailman3_core_api_hostname: "{{ inventory_hostname }}"
mailman3_core_api_admin_user: adminuser
mailman3_core_api_admin_pass: adminpass
mailman3_archiver_key: archiverkey
mailman3_config:
  mailman:
    site_owner: listmaster@example.org
  database:
    class: mailman.database.postgresql.PostgreSQLDatabase
    url: postgresql://mailman3_core:mypassword@mailman.example.org/mailman3_core
mailman3_django_config:
  haystack_engine: haystack.backends.elasticsearch7_backend.Elasticsearch7SearchEngine
  secret_key: supersecret
  # Don't define hyperkitty_attachment_folder. If not set, it will store attachments in the db.
  # hyperkitty_attachment_folder: "{{ mailman3_web_var_dir }}/attachments"
  csrf_trusted_origins:
    - http://mailman.example.org
    - https://mailman.example.org
  databases:
    default:
      ENGINE: 'django.db.backends.postgresql_psycopg2'
      NAME: 'mailman3_web'
      HOST: 'mailman.example.org'
      USER: 'mailman3_web'
      PASSWORD: 'mypassword'
      PORT: '5432'
mailman3_extra_packages:
  - psycopg2-binary
mailman3_nginx_ssl_certs: |
  include snippets/snakeoil.conf;
      # ssl_certificate /etc/letsencrypt/live/{{ mailman3_web_url }}/fullchain.pem;
      # ssl_certificate_key /etc/letsencrypt/live/{{ mailman3_web_url }}/privkey.pem;
mailman3_sasl_passwd: smtp.mailgun.org postmaster@mailman.example.org:555555555
mailman3_relayhost: smtp.mailgun.org:587
```

/etc/ansible/host_vars/database.example.org/vars:  

```
mailman3_database_list:
  - name: mailman3_web
    username: mailman3_web
    password: mypassword
    subnet: "0.0.0.0/0"
  - name: mailman3_core
    username: mailman3_core
    password: mypassword
    subnet: "0.0.0.0/0"
```

Similarly, an example with a unix socket to reach the database. Notice the database username change:    

```yaml
mailman3_django_superusers:
  - name: admin
    pass: admin
    email: admin@example.org
mailman3_core_api_admin_user: adminuser
mailman3_core_api_admin_pass: adminpass
mailman3_archiver_key: archiverkey
mailman3_config:
  mailman:
    site_owner: listmaster@example.org
  database:
    class: mailman.database.postgresql.PostgreSQLDatabase
    url: postgres:///mailman3_core?host=/var/run/postgresql
mailman3_django_config:
  secret_key: supersecret
  # Don't define hyperkitty_attachment_folder. If not set, it will store attachments in the db.
  # hyperkitty_attachment_folder: "{{ mailman3_web_var_dir }}/attachments"
  csrf_trusted_origins:
    - http://mailman.example.org
    - https://mailman.example.org
  databases:
    default:
      ENGINE: django.db.backends.postgresql_psycopg2
      NAME: mailman3_web
      HOST: /var/run/postgresql
mailman3_extra_packages:
  - psycopg2-binary
mailman3_nginx_ssl_certs: |
  include snippets/snakeoil.conf;
      # ssl_certificate /etc/letsencrypt/live/{{ mailman3_web_url }}/fullchain.pem;
      # ssl_certificate_key /etc/letsencrypt/live/{{ mailman3_web_url }}/privkey.pem;
mailman3_sasl_passwd: smtp.mailgun.org postmaster@mailman.example.org:555555555
mailman3_relayhost: smtp.mailgun.org:587
mailman3_database_list:
  - name: mailman3_web
    # With a unix socket the db username should match the linux username, which is often "mailman3-web".  
    username: mailman3-web
    password: mypassword
    subnet: "0.0.0.0/0"
  - name: mailman3_core
    username: mailman3_core
    password: mypassword
    subnet: "0.0.0.0/0"
```

The next example comes from the original galaxyproject role. A more complex setup with multiple list domains, Xapian haystack engine, distribution of Postfix lmtp maps to MX
servers, and DKIM:

```yaml
- name: Install Mailman 3
  hosts: all
  vars:
    mailman3_domains:
      - lists.example.org
      - lists.example.com
    mailman3_django_superusers:
      - name: admin
        pass: admin
        email: admin@example.org
    mailman3_core_api_admin_user: adminuser
    mailman3_core_api_admin_pass: adminpass
    mailman3_archiver_key: archiverkey

    mailman3_config:
      mailman:
        site_owner: listmaster@example.org
      mta:
        # bypass the content filter (amavisd-new) for Mailman messages since they were scanned on the way in
        smtp_port: 10025
      database:
        class: mailman.database.postgresql.PostgreSQLDatabase
        url: postgres:///mailman3_core?host=/var/run/postgresql
      arc:
        enabled: 'yes'
        privkey: "/etc/dkimkeys/mail.private"
        selector: mail
        domain: lists.example.org
        dmarc: 'yes'
        dkim: 'yes'
        authserv_id: lists.example.org
        trusted_authserv_ids: lists.example.org, lists.example.com, example.org

    mailman3_django_config:
      secret_key: supersecret
      use_x_forwarded_host: true
      hyperkitty_attachment_folder: "{{ mailman3_web_var_dir }}/attachments"
      databases:
        default:
          ENGINE: django.db.backends.postgresql_psycopg2
          NAME: mailman3_web
          HOST: /var/run/postgresql
      haystack_engine: xapian_backend.XapianEngine

    mailman3_extra_packages:
      - psycopg2-binary
      - xapian-haystack

    mailman3_distribute_maps:
      - host: all
        mailman3_distribute_maps_dir: "{{ mailman3_distribute_maps_dir }}"
        mailman3_postmap_command: /usr/sbin/postmap
      - host: mx1.example.org
      - host: mx2.example.org
      - host: mx3.example.org
```

License
-------

MIT

Author Information
------------------

The [CPPAlliance](https://cppalliance.org)  
The [Galaxy Community](https://galaxyproject.org/) and [contributors](https://github.com/galaxyproject/ansible-mailman3/graphs/contributors)
