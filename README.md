[![ansible-galaxy gimoh.docker_container](https://img.shields.io/badge/ansible--galaxy-gimoh.docker__container-brightgreen.svg)](https://galaxy.ansible.com/list#/roles/3101) [![License GPLv3](https://img.shields.io/badge/license-GPLv3-blue.svg)](http://www.gnu.org/licenses/gpl-3.0.html)

Role Name
=========

Role for managing dockerized services.

It wraps the docker module and provides the following features on top of it:

- automatically creates a data volume container if any volumes are specified
- allows marking the container to be automatically registered with an
  nginx reverse proxy provided by the
  [jwilder/nginx-proxy](https://registry.hub.docker.com/u/jwilder/nginx-proxy/)
  docker image when it's running
- uploads SSL/TLS certificate for the service for use with the above mentioned
  nginx image

The role
[gimoh.docker_nginx_proxy](https://galaxy.ansible.com/list#/roles/3097) can be
used to set up the jwilder/nginx-proxy container.

Requirements
------------

- [Docker](https://www.docker.com/), I personally use the role
  [angstwad.docker_ubuntu](https://galaxy.ansible.com/list#/roles/292) to
  deploy it
- [jwilder/nginx-proxy](https://registry.hub.docker.com/u/jwilder/nginx-proxy/)
  based container if you want to use the reverse-proxy feature to expose a
  provided service ouside the host; this is completely optional

NOTE: this role requires an implementation of a Jinja filter for recursively
merging dicts (hashes).  There is currently a
[pull request pending in Ansible](https://github.com/ansible/ansible/pull/7872)
to add such a filter, but in the meantime you can download the
[hash_merge.py file](https://github.com/darkk/ansible/raw/merge_hash/lib/ansible/runner/filter_plugins/hash_merge.py)
and place it in a `filter_plugins/` directory relative to your top-level
playbook.

Role Variables
--------------

_role parameters_

```
# Any parameters to be passed to the docker module.
opts: {}

# FQDN the service should be exposed at.
#
# Only if you want to use the nginx-proxy feature.  Note that you need to
# configure your DNS to forward the specified FQDN to the docker host, e.g.
# with a CNAME record.
proxy_virtual_host: none

# Optional additional nginx configuration snippet to inject into vhost
# configuration for the service.
proxy_config: none

# Enable uploading of SSL/TLS certificate for the service for use with the
# reverse proxy.
proxy_tls: false
```

_defaults/main.yml_

```
# Path to directory *on the machine where Ansible runs* where SSL/TLS
# certificates for the exposed services can be found.
#
# Only needed if using the nginx-proxy feature and if you want the reverse
# proxy to provide SSL/TLS termination for the service (and of course if the
# certs aren't stored in the playbook directory).
dc_src_certs_dir: .

# Common with `gimoh.docker_nginx_proxy` role.  Specifies path to directory
# where SSL/TLS certificates for the exposed services will be uploaded to.
#
# Only used if using nginx-proxy feature and proxy_tls is enabled.
dnp_nginx_certs_dir: /etc/pki/svc-certs

# Common with `gimoh.docker_nginx_proxy` role.  Specifies path to
# directory where nginx vhost config snippets will be stored.
#
# Only used if using nginx-proxy feature and proxy_config is defined.
dnp_nginx_vhost_dir: /etc/nginx-proxy-vhost.d
```

_vars/main.yml_

```
# These are internal implementation details, they are assigned on each
# invocation of the role based on parameters passed to the role.
dc_docker_opts: "{{ opts|default({}) }}"
dc_name: "{{ dc_docker_opts.get('name', 'unnamed') }}"
dc_proxy_tls: "{{ proxy_tls|default(False) }}"
```

_facts set by this role_

```
# This role creates a `docker_container` fact which is a dict (hash) mapping
# container names to the data returned by `docker inspect` (specifically to
# the fact data for the container that's set by the `docker` module).
#
# This makes it possible to use data from any container managed earlier in the
# play instead of just the last one (provided by the `docker_containers` fact
# set by the `docker` module).
#
# NOTE that `name` here is the name `docker run` sets for the container
# (either generated or specified via `name` parameter), which means it's
# preceeded by a `/`.  I.e. even if `name: foo` is specified, the key will
# be `/foo`.
docker_container[name] = { (docker inspect data) }
```

Dependencies
------------

- [gimoh.docker_nginx_proxy](https://galaxy.ansible.com/list#/roles/3097)
  (optional) only if you want to use the nginx-proxy feature, no variables
  need to be set, uses `dnp_nginx_certs_dir` if `proxy_tls` is enabled

Example Playbook
----------------

Basic usage (equivalent to using the `docker` module directly):

    - hosts: servers
      roles:
        - { role: gimoh.docker_container, opts: { image: gimoh/sleeping-beauty } }

To explicitly name the container (if not specified docker will generate a
random name, but Ansible output will refer to it as _unnamed_, e.g. in task
names).

Also note that the `docker` module matches containers by either name, or
image + tag + command, so specifying name is recommended unless using `count`.

    - hosts: servers
      roles:
        - role: gimoh.docker_container
          opts: { image: gimoh/sleeping-beauty, name: test1 }

This demonstrates different ways of passing volumes.

Note that `volumes` here is a parameter **to this role** instead of the
`docker` module (i.e. it isn't specified in `opts` parameter), this way a
docker data volume container will be created for each of the containers
specified (named `test2-vol` and `test3-vol` respectively) and the main
containers will be created with `volumes_from` pointing to their data volume
container.  The data volume containers are created from
[tianon/true](https://registry.hub.docker.com/u/tianon/true/) image.

    - hosts: servers
      roles:
        - role: gimoh.docker_container
          opts: { image: gimoh/sleeping-beauty, name: test2 }
          volumes: ['/tmp', '/srv']
        - role: gimoh.docker_container
          opts: { image: gimoh/sleeping-beauty, name: test3 }
          volumes: '/tmp:/host-tmp:ro'

This demonstrates marking container to be registered with the nginx reverse
proxy provided by the
[jwilder/nginx-proxy](https://registry.hub.docker.com/u/jwilder/nginx-proxy/)
docker image when it's running.

The container will be created with an `env` variable
`VIRTUAL_HOST={{ proxy_virtual_host }}` injected into the other passed in
options in `opts`.  Also an nginx configuration file will be
automatically created with the contents as passed to the
``proxy_config`` option.

    - hosts: servers
      roles:
        - role: gimoh.docker_container
          opts: { image: gimoh/sleeping-beauty, name: test4 }
          proxy_virtual_host: test4.f.q.d.n
          proxy_config: >
            server_tokens off;
            client_max_body_size 100m;

And finally a more complete example, including passing more complex parameters
to the `docker` module and using the facts set by this role:

    - hosts: servers
      roles:
        - role: gimoh.docker_container
          opts:
            image: gimoh/sleeping-beauty
            name: test5
            env:
              FOO: bar
          volumes:
            - '/tmp'
            - '/data:/srv'

        - role: gimoh.docker_container
          opts:
            image: gimoh/sleeping-beauty
            name: test6
            env:
              BACKEND_IP: >
                {{ docker_container['/test5'].NetworkSettings.IPAddress }}
          proxy_virtual_host: test6.f.q.d.n
          proxy_ssl: true

License
-------

GPLv3

Author Information
------------------

Contact me through GitHub issues, etc.

gimoh
