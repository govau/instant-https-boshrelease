# Instant HTTPS BOSH Release

*We have archived this repository as it is no longer actively used or maintained by the DTA.*

This BOSH release is designed as a simple, low-maintenance way to just add HTTPS to web services.  It's primarily designed for dashboards, status pages and other tooling used by dev and ops teams, but might be useful for other purposes.

## How it Works
It's simply a [Caddy webserver](https://caddyserver.com/) instance that reverse proxies to your service.  SSL certificates are automatically provided by [Let's Encrypt](https://letsencrypt.org/) (or another ACME-compatible provider).

The release can be added to existing deployments, and the proxy can run on the same machine as the service.  This is the recommended setup because no unencrypted traffic will leave the machine.  It's also possible to run a standalone proxy that proxies to services running elsewhere on your network.

## Important Info About Let's Encrypt
If you use this with one of Let's Encrypt's API URLs, you'll be agreeing to the [Let's Encrypt Subscriber Agreement](https://letsencrypt.org/repository/).

By default, the configuration uses Let's Encrypt's staging API's URL (https://acme-staging.api.letsencrypt.org/directory).  Let's Encrypt's production API has a rate limit of 60 failed validations per hour, so it's recommended that you leave this setting until you're satisfied your deployment works.  (The staging API works just like the production one, but returns a certificate that's not trusted by browsers.)  To get production certs from Let's Encrypt, set the `acme_url` job property to `https://acme-v01.api.letsencrypt.org/directory`.

## Usage
### Standalone Proxy
It's recommended to use the all-in-one setup (below) if possible.

Here's an example deployment manifest:
```yaml
---
name: instant-https-standalone

releases:
  - name: instant-https
    url: https://github.com/govau/instant-https-boshrelease/releases/download/v0.4.0/instant-https-0.4.0.tgz
    sha1: f603d51ce64efb0145a2842245404946782b1827
    version: latest

stemcells:
  - alias: default
    os: ubuntu-trusty
    version: latest

instance_groups:
  - name: https_proxy
    instances: 1
    azs: [z1]
    networks:
      - name: default
    jobs:
      - name: proxy
        release: instant-https
        properties:
          hostname: dashboard.example.com
          contact_email: webmaster@example.com
          # Set this for real production certificates:
          # acme_url: https://acme-v01.api.letsencrypt.org/directory
          backends:
            - "192.168.0.1:8000"  # Host and port of existing dashboard
    vm_type: default
    stemcell: default

update:
  canaries: 1
  max_in_flight: 1
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
```

### All-in-one
It should be possible to make an all-in-one deployment by tweaking an existing, unprotected deployment.

* Add the the release
* Configure the service to listen on localhost and some port that's not 80 or 443
* Add and configure the proxy job

```yaml
---
name: my-foo-dashboard-instance

releases:
  - name: foo-dashboard
    version: latest
  - name: instant-https  # Add the release
    url: https://github.com/govau/instant-https-boshrelease/releases/download/v0.4.0/instant-https-0.4.0.tgz
    sha1: f603d51ce64efb0145a2842245404946782b1827
    version: latest

stemcells:
  - alias: default
    os: ubuntu-trusty
    version: latest

instance_groups:
  - name: https_proxy
    instances: 1
    azs: [z1]
    networks:
      - name: default
    jobs:
      - name: dashboard
        release: foo-dashboard
        properties:
          listen: 127.0.0.1  # Configure the bind ip and port of the service
          port: 8000
      - name: proxy  # Add and configure the proxy job
        release: instant-https
        properties:
          hostname: dashboard.example.com
          contact_email: webmaster@example.com
          # Set this for real production certificates:
          # acme_url: https://acme-v01.api.letsencrypt.org/directory
          backends:
            - "127.0.0.1:8000"
    vm_type: default
    stemcell: default

update:
  canaries: 1
  max_in_flight: 1
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
```

### Links Support
BOSH 2.0 links are also supported.  The `proxy` optionally consumes a link of type `conn`, named `backend`.  See [the BOSH links docs](https://bosh.io/docs/links.html) for more information.

```yaml
instance_groups:
  - name: dashboard
    instances: 1
    azs: [z1]
    networks:
      - name: default
    vm_type: default
    stemcell: default
    jobs:
      - name: dashboard
        release: foo-dashboard
        properties:
          port: 8000
        provides:
          dashboard: {as: backend_link}  # Provide backend link here
  - name: proxy
    instances: 1
    azs: [z1]
    networks:
      - name: default
    vm_type: default
    stemcell: default
    jobs:
      - name: proxy
        release: instant-https
        properties:
          hostname: dashboard.example.com
          contact_email: webmaster@example.com
          default_backend_port: 8000
        consumes:
          backend: {from: backend_link}  # Hook link to proxy here (matching using as/from)
```

The link `name` doesn't need to match if `as` and `from` are used as in the example above, but the link `type` does.  Because the `type` is just a meaningless, unstandardised tag, it's likely that jobs from different releases won't match.  As a hacky workaround until BOSH's links provide a better solution, you can run another no-op job in the instance group you want to link to:

```yaml
instance_groups:
  - name: dashboard
    instances: 1
    azs: [z1]
    networks:
      - name: default
    vm_type: default
    stemcell: default
    jobs:
      - name: dashboard
        release: foo-dashboard
        properties:
          port: 8000
      - name: links-kludger  # This job can provide a link of the right type
        release: instant-https
        provides:
          backend: {as: backend_link}
  - name: proxy
    instances: 1
    azs: [z1]
    networks:
      - name: default
    vm_type: default
    stemcell: default
    jobs:
      - name: proxy
        release: instant-https
        properties:
          hostname: dashboard.example.com
          contact_email: webmaster@example.com
          default_backend_port: 8000
        consumes:
          backend: {from: backend_link}  # BOSH should happily accept this link
```
