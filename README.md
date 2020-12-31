# ansible-role-demostat

Ansible role to install and configure [demostat](https://demostat.de).

## Requirements

- [Python3](https://www.python.org/) with [venv support](https://docs.python.org/3/library/venv.html)
- Systemd
- A functioning database
  - Postgresql is recommended, this role only has been tested with postgresql
  - See geelingguys [ansible-role-postgresql](github.com/geerlingguy/ansible-role-postgresql) for a role to setup and configure postgresql
- Root access to the machine running debian or derivates
- A Domain

## Role variables

Available variables are listed below together with their default values as specified in `defaults/main.yml`.

### Generic environment configuration

```yml
demostat_user: "demostat"
demostat_group: "{{ demostat_user }}"
demostat_path: "/opt/demostat"
demostat_venv_path: "{{ demostat_path }}/venv"
```

Where to place the demostat files and configuration, who should be the owner and where to place the python virtual-env.

```yml
demostat_systemd_prefix: 'demostat-'
```

Prefix for the installed gunicorn service (e.g. `demostat-gunicorn.service`)

```yml
demostat_apt_update: true
```

Update the apt-cache while installing demostat dependencies. Especially needed for older docker containers or outdated cloud images.

```yml
demostat_dependencies:
  - build-essential
  - memcached
  - python3-dev
  - python3-wheel
  - python3-setuptools
  - python3-virtualenv
  - memcached
  - libmemcached-dev
```

Apt packages that are required for demostat and the installation.

```yml
demostat_secret_key: null
```

Secret key for [securing the application](https://docs.djangoproject.com/en/2.2/ref/settings/#std:setting-SECRET_KEY), _must_ be set.

```yml
demostat_debug: false
```

Debug enables stack traces on errors.

```yml
demostat_db_engine: 'sqlite3'  # mysql, sqlite3 or postgresql_psycopg2
demostat_db_name: 'demostat.sqlite3'
demostat_db_user: ''
demostat_db_pass: ''
demostat_db_host: ''
demostat_db_port: ''
```

Database connectivity for demostat, directly maps to the django settings: <https://docs.djangoproject.com/en/2.2/ref/settings/#std:setting-DATABASES>. Postgresql (`postgresql_psycopg2`) is, due to its superior performance, recommended in production. SQLite3 should only be used for testing purposes.

```yml
demostat_email_backend: ''
demostat_email_host: ''
demostat_email_port: ''
demostat_email_host_user: ''
demostat_email_host_pass: ''
demostat_email_default_from_email: '{{ demostat_email_host_user }}'
demostat_email_confirmation_from: '{{ demostat_email_default_from_email }}'
demostat_email_server_email: '{{ demostat_email_default_from_email }}'
demostat_email_use_ssl: true
```

Configuration on how demostat will send E-Mail. See <https://docs.djangoproject.com/en/2.2/topics/email/#email-backends> for additional backends and how to configure them.

```yml
demostat_time_zone: '{{ ansible_date_time["tz"] }}'
```

The timezone used by demostat, defaults to the systems current timezone.

_Note for Regions with summer-/wintertime:_ It is advised to set the timezone manually since ansible does not provide the location of the timezone (e.g. `Europe/Berlin`) but the timezone according to the offset (CET with `+01:00` in the winter and CEST with `+02:00` in the summer). Therefore, you would have to re-run this role after each time change.

```yml
demostat_allowed_hosts:
  - 'localhost'
  - '127.0.0.1'
  - '::1'
  - '::ffff::127.0.0.1'
  - '{{ demostat_nginx_server_name }}'
```

Hosts and domain names django can serve content for, if a request redirected to django does not match this list, the request will be rejected.

```yml
demostat_admins:
  - name: Demostat Admin
    email: demostat@localhost
```

Errors in the Django-Application will be sent to the listed E-Mails: <https://docs.djangoproject.com/en/2.2/ref/settings/#admins>

```yml
demostat_cache_backend: 'django.core.cache.backends.memcached.PyLibMCCache'
demostat_cache_location: 'localhost'
```

The cache used for all sorts of django-caches. By default it usese memcached which is also installed by this role. See the django documentation for possible settings: <https://docs.djangoproject.com/en/2.2/ref/settings/#std:setting-CACHES>

[Gunicorn](https://gunicorn.org/) is the WSGI HTTP server that runs demostat.

```yml
demostat_gunicorn_listen: '127.0.0.1'
demostat_gunicorn_port: 8000
```

IP and Port to bind to.

```yml
demostat_gunicorn_procname: '{{ demostat_systemd_prefix }}gunicorn'
```

Title of the process, useful for differentiating multiple gunicorn processes.

```yml
demostat_gunicorn_workers: 4
```

Amount of worker-processes to handle requests.

### 3rd Party Helpers

This role is able to do some basic configuration of nginx. Those configurations are _not especially secure_ nor complete at all. It just aims to help out a bit.


```yml
demostat_nginx_enabled: false
demostat_nginx_server_name: 'demostat.example.com'
demostat_nginx_ssl: true
demostat_nginx_path: '/etc/nginx/sites-enabled/demostat.conf'
demostat_nginx_ssl_redirect: true
demostat_nginx_v6only: false
demostat_nginx_ssl_certificate: null
demostat_nginx_ssl_key: null
```

Set `demostat_nginx_enabled` to `true` in order to install a nginx configuration to the specified location and restart nginx afterwards.

_Note_: This role will neither install nginx nor generate SSL-Certificates from e.g. letsencrypt for you. That's on you. You can find some roles for that in the [related projects](#related-projects) section.

## Example Playbook

This is a small sample playbook utilizing most of the roles features. To run this playbook you need to have the postgresql database available. The lines ending in `# Change` contain secrets which should be changed in production. Same goes for domain names, E-Mail contacts and so on.

```yml
- hosts: demostat
  become: true
  vars_file:
    - vars/demostat.yml
  roles:
    - e1mo.demostat
```

_Inside `vars/demostat.yml`_

```yml
demostat_siteowner: 'postmaster@mydomain.com'

demostat_secret_key: 'fooSecretBar'  # Change

demostat_db_engine: 'postgresql_psycopg2'
demostat_db_name: 'demostat'
demostat_db_user: 'demostat'
demostat_db_pass: 'no-less-secret'  # Change
demostat_db_host: 'localhost'

demostat_email_default_from_email: 'noreply@demostat.example.com'

demostat_time_zone: 'Europe/Berlin'

demostat_admins:
  - name: Fellow Hacker
    email: noc@hacker.org

demostat_archiver_key: 'not-leaking-that-one'  # Change

# You need to have nginx installed before hand
demostat_nginx_enabled: true
demostat_nginx_path: '/etc/nginc/sites-enabled/lists.mydomain.com.conf'
demostat_nginx_server_name: 'demostat.example.com'

demostat_nginx_ssl_certificate: '/etc/ssl/certs/ssl-cert-snakeoil.pem'
demostat_nginx_ssl_key: '/etc/ssl/private/ssl-cert-snakeoil.key'
```

## License

GNU General Public License v3.0 or later. See [LICENSE](LICENSE) for the full license.

## Related projects

- The [demostat project](https://demostat.de/)
- The [ansible](https://github.com/ansible/ansible) project itself, without which this role could not exist.
- [ansible-role-postgresql](https://github.com/geerlingguy/ansible-role-postgresql) from [Jeff Geerling aka. geerlingguy](https://github.com/geerlingguy) which can install and configure postgresql easily.
- [ansible-nginx-base](https://github.com/rixx/ansible-nginx-base) and [ansible-dehydrated](https://github.com/rixx/ansible-dehydrated) (get letsencrypt certs) by [rixx](https://rixx.de/) are both really simple to work with and don't get in your way. I have patched and updated both roles a fair bit, just haven't submitted the patches: [ansible-nginx-base](https://github.com/e1mo/ansible-nginx-base) and [ansible-dehydrated](https://github.com/e1mo/ansible-dehydrated).

## Author information

Written by [Moritz 'e1mo' Fromm](https://e1mo.de) for the [demostat project](https://demostat.de/).
