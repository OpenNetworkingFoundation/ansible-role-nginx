<!--
SPDX-FileCopyrightText: © 2020 Open Networking Foundation <support@opennetworking.org>
SPDX-License-Identifier: Apache-2.0
--!>
# nginx

NGINX Web Server

## Requirements

Minimum tested Ansible version: 2.9.5

## What this role does

This runs an nginx web server, which can be configured to to:

1. Serve static files
2. Redirect traffic to another location
3. Reverse proxy for another service

It also can provide TLS termination (and a 301 redirect from http:// to
https:// URLs if TLS is enabled), as well as providing HTTP Basic
Authentication support.

The TLS configuration settings are designed to qualify for a minimum of an *A*
rating (as of 2020-07-04) on the [Qualsys SSL Labs server
test](https://www.ssllabs.com/), using the recommendations from the [Mozilla
SSL configuration generator](https://ssl-config.mozilla.org/).

HTTP Basic authentication covers the entire root (`/`) of the site, and can
also apply to proxied connections.

To allow LetsEncrypt and other ACME based certificate issuing programs to
function, the `http://<anything>/.well-known/acme-challenge` uri on all hosts
is served from a directory specified by the `acme_challenge_dir` variable. This
also works for redirects, basic authentication required, and reverse proxied
sites.

If the role or configuration has changed and the templated files are changed
when the role is re-run, backups of the configuration files will be generated
on the target host.

Before restarting the `nginx` daemon on the host system, the entire
configuration will be checked by running `nginx -t` by the `test-nginx-config`
handler. If this handler fails, it will give the running admin a chance to fix
any configuration problems prior to restarting the daemon, but will leave the
configuration in a broken state.

## What this role doesn't do

This is an opinionated role and only handles the most common use cases. It does
not handle:

- Every configuration option available in nginx. The [official nginx
  role](https://github.com/nginxinc/ansible-role-nginx) has many more
  configuration options exposed, at a cost of being much more complicated to
  configure.

- Complicated redirect rules. The grammar for these was too complicated to make
  into a general case, so if they're required use the `extra_config` which is
  loaded into the `server {}` configuration scope, and specify rules using the
  [rewrite module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html)

- Wildcard matching of domains. These were generally done to host many sites
  behind one TLS certificate, which is less needed now that we have
  LetsEncrypt.  Define site aliases instead, setting `cert_name` and use SAN to
  generate a multi-domain certificate.

## How do I obtain certificates for many domains?

This is a bit of a chicken/egg problem. Here's one solution:

1. Run the `acme` role with no config to create the challenge dir and user
   accounts that will own that dir.

2. Run this role without TLS configured in the variable files, to create
   virtualhosts that can obtain http-01 challenges.

3. Run the `acme` role to obtain and install LetsEncrypt certificate for just
   the `cert_name` domain.

4. Configure TLS for all domains then rerun this role

5. Rerun the `acme` role to obtain and install certificates for all the
   domains. It will restart `nginx`.

Or, use DNS based challenges with `acme`, which don't depend on nginx serving
the challenges.

## Defaults

There are several global config options, which apply to all sites being hosted.

``` yaml
# http://nginx.org/en/docs/ngx_core_module.html#worker_processes
nginx_conf_worker_processes: "auto"

# http://nginx.org/en/docs/ngx_core_module.html#worker_connections
nginx_conf_worker_connections: 512

# http://nginx.org/en/docs/ngx_core_module.html#multi_accept
nginx_conf_multi_accept: "off"

# http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size
nginx_conf_client_max_body_size: "1m"

# location of TLS certificates
certificate_dir: "/etc/acme/certs"

# location of ACME Challenges (LetsEncrypt)
acme_challenge_dir: "/srv/acme"

# ACME software user name for setting permissions on challenge dir
# NOTE: this will cause failures if you haven't previously run the the acme
# role, so override it when that role isn't being used.
acme_username: "acme"

# block specific user agents (webscrapers)
blocked_user_agents: "DotBot|MJ12bot|SemrushBot|PetalBot|AhrefsBot"
```

The `vhosts` is used to define virtualhosts on the nginx server. This is a list
of all available options:

``` yaml
vhosts:
  - name: site.example.com  # name of the server being used.
    tls: true # whether or not to enable TLS. Requires that certificates be created.
    cert_name: static.example.com  # used to specify the filepath to the TLS certificate, if a multi-domain SAN cert is used. Default is to use the `name` value
    owner: root # username of owner of static files for this virtualhost, used to delegate the ability to update a static site.
    insecure_port: 80 # default http:// port
    secure_port: 443 # default https:// port
    auth_scope: "mysite"  # which item in auth_scopes to use for passwords. Setting this enables HTTP Basic Auth.
    proxy_pass: "http://localhost:8080"  # set this to the URL to reverse proxy traffic to another service
    custom_root: "/path/to/site/root"  # set this to serve files from a nonstandard (not /srv/sites/<url>) location
    autoindex: false # whether or not to automatically generate an index in directories that lack an index.html file
    aliases:  # list of domain aliases to redirect
      - static.example.com
      - www.example.com
    redirect_url: https://google.com  # destination for 301 redirection of the aliases
    strip_request_uri: true  # Set to true to strip off the path at the end of the URI when redirecting
    extra_config: |  # Extra arbitrary configuration to put inside the server {} block
      <arbitrary configuration>
```

To provide HTTP basic auth, you can create an item in `auth_scopes`, which
gives a scope and lists of usernames and password.

This configuration is separate from the vhosts block so scopes can be reused on
multiple sites, and so that passwords can be can be stored in a separate
configuration file.  The passwords are encrypted using [salted
SHA1](http://nginx.org/en/docs/http/ngx_http_auth_basic_module.html#auth_basic_user_file).

``` yaml
auth_scopes:
  - scope: mysite
    users:
    - name: ghopper
      password: verysecurepassword
    - name: dknuth
      password: anotherpassword
  - scope: othersite
    users:
    - name: aturing
      password: yetanotherpw
```

### Example: Static site without TLS enabled

``` yaml
vhosts:
  - name: site.example.com
```

### Example: Static site with TLS enabled

> NOTE: Must create TLS certificates outside of this role. The recommended role
> to do this is the `acme` role.

``` yaml
vhosts:
  - name: site.example.com
    tls: true
```

### Example: Proxy site with TLS enabled

``` yaml
vhosts:
  - name: site.example.com
    tls: true
    proxy_pass: "http://localhost:8080"
```

## Example Playbook

```yaml
- hosts: all
  vars:
  - vhosts:
    - name: site.example.com
  roles:
    - nginx
```

## Testing

There are many tests in the `molecule/` directory, covering static, autoindexed
authenticated, redirect, and proxy examples.

If you want to add functionality to this role, please write tests to cover that
functionality.

## License and Author

© 2020 Open Networking Foundation <info@opennetworking.org>

License: Apache-2.0

`files/ffdhe2048.txt` was obtained from the [Mozilla SSL configuration
generator website](https://ssl-config.mozilla.org/) and appears to be under
MPL-2.0, as it came from the
[mozilla/ssl-config-generator](https://github.com/mozilla/ssl-config-generator/)
repo.
