# Instant HTTPS BOSH Release
This BOSH release is designed as a simple, low-maintenance way to just add HTTPS to web services.  It's primarily designed for dashboards, status pages and other tooling used by dev and ops teams, but might be useful for other purposes.

## How it Works
It's simply a [Caddy webserver](https://caddyserver.com/) instance that reverse proxies to your service.  SSL certificates are automatically provided by [Let's Encrypt](https://letsencrypt.org/) (or another ACME-compatible provider).

The release can be added to existing deployments, and the proxy can run on the same machine as the service.  This is the recommended setup because no unencrypted traffic will leave the machine.  It's also possible to run a standalone proxy that proxies to services running elsewhere on your network.

## Usage
### Standalone Proxy
It's recommended to use the all-in-one setup (below) if possible.

Here's an example deployment manifest:
```yaml
---
name: instant-https-standalone

releases:
  - name: instant-https
    url: https://github.com/govau/instant-https-boshrelease/releases/download/v0.1.0/instant-https-1.tgz
    sha1: 5430715589f418321c2d6daf187cce1ea4a51fd2
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
            - 192.168.0.1:8000  # Host and port of existing dashboard
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
    url: https://github.com/govau/instant-https-boshrelease/releases/download/v0.1.0/instant-https-1.tgz
    sha1: 5430715589f418321c2d6daf187cce1ea4a51fd2
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
        release foo-dashboard
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
            - 127.0.0.1:8000
    vm_type: default
    stemcell: default

update:
  canaries: 1
  max_in_flight: 1
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
```
