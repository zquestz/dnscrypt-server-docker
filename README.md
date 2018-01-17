[![DNSCrypt](https://raw.github.com/jedisct1/dnscrypt-server-docker/master/dnscrypt-small.png)](https://dnscrypt.org)

DNSCrypt server Docker image
============================

Run your own caching, non-censoring, non-logging, DNSSEC-capable,
[DNSCrypt](http://dnscrypt.org)-enabled DNS resolver virtually anywhere!

If you are already familiar with Docker, it shouldn't take more than 5 minutes
to get your resolver up and running.

Installation
============

Think about a name. This is going to be part of your DNSCrypt provider name.
If you are planning to make your resolver publicly accessible, this name will
be public.
It has to look like a domain name (`example.com`), but it doesn't have to be
a registered domain.

Let's pick `example.com` here.

Download, create and initialize the container, once and for all:

    $ docker run --name=dnscrypt-server -p 443:443/udp -p 443:443/tcp --net=host \
        jedisct1/unbound-dnscrypt-server init -N example.com

This will only accept connections via DNSCrypt on the standard port (443).

`--net=host` provides the best network performance, but may have to be
removed on some shared containers hosting services.

Now, to start the whole stack:

    $ docker start dnscrypt-server

Done.

To check that your DNSCrypt-enabled DNS resolver is accessible, run the
DNSCrypt client proxy on another host:

    # dnscrypt-proxy \
        --provider-key=<provider key, as displayed when the container was initialized> \
        --resolver-address=<dnscrypt resolver public IP address> \
        --provider-name=2.dnscrypt-cert.example.com

And try using `127.0.0.1` as a DNS resolver.

Note that the actual provider name for DNSCrypt is `2.dnscrypt-cert.example.com`,
not just `example.com` as initially entered. The full name has to start with
`2.dnscrypt-cert.` for the client and the server to use the same version of the
protocol.

Let the world know about your server
====================================

Is your brand new DNS resolver publicly accessible?

Fork the [dnscrypt-proxy repository](https://github.com/jedisct1/dnscrypt-proxy),
edit the [dnscrypt.csv](https://github.com/jedisct1/dnscrypt-proxy/blob/master/dnscrypt-resolvers.csv)
file to add your resolver's informations, and submit a pull request to have it
included in the list of public DNSCrypt resolvers!

Customizing Unbound
============

To add new configuration to Unbound, add files to the `/opt/unbound/etc/unbound/zones`
directory. All files ending in `.conf` will be processed. In this manner, you
can add any directives to the `server:` section of the Unbound configuration.

Serve custom DNS records on a local network
------------------------------------------
While Unbound is not a full authoritative name server, it supports resolving
custom entries in a way that is serviceable on a small, private LAN. You can use
unbound to resolve private hostnames such as `my-computer.example.com` within
your LAN.

To support such custom entries using this image, first map a volume to the zones
directory. Add this to your `docker run` line:

    -v /myconfig/zones:/opt/unbound/etc/unbound/zones

The whole command to create and initialize a container would look something like
this:

    $ docker run --name=dnscrypt-server \
        -v /myconfig/zones:/opt/unbound/etc/unbound/zones \
        -p 443:443/udp -p 443:443/tcp --net=host \
        jedisct1/unbound-dnscrypt-server init -N example.com

Create a new `.conf` file:

    $ touch /myconfig/zones/example.conf

Now, add one or more unbound directives to the file, such as:

    local-zone: "example.com." static
    local-data: "my-computer.example.com. IN A 10.0.0.1"
    local-data: "other-computer.example.com. IN A 10.0.0.2"

Troubleshooting
---------------

If Unbound doesn't like one of the newly added directives, it
will probably not respond over the network. In that case, here are some commands
to work out what is wrong:

    $ docker logs dnscrypt-server
    $ docker exec dnscrypt-server /opt/unbound/sbin/unbound-checkconf

Details
=======

- Caching resolver: [Unbound](https://www.unbound.net/), with DNSSEC, prefetching,
and no logs. The number of threads and memory usage are automatically adjusted.
Latest stable version, compiled from source. qname minimisation is enabled.
- [LibreSSL](http://www.libressl.org/) - Latest stable version, compiled from source.
- [libsodium](https://download.libsodium.org/doc/) - Latest stable version,
minimal build compiled from source.
- [dnscrypt-wrapper](https://github.com/Cofyc/dnscrypt-wrapper) - Latest stable version,
compiled from source.
- [dnscrypt-proxy](https://github.com/jedisct1/dnscrypt-proxy) - Latest stable version,
compiled from source.

Keys and certificates are automatically rotated every 12 hour.

Kubernetes
==========

Kubernetes configurations are located in the `kube` directory. Currently these assume
a persistent disk named `dnscrypt-keys` on GCE. You will need to adjust the volumes
definition on other platforms. Once that is setup, you can have a dnscrypt server up
in minutes.

* Create a static IP on GCE. This will be used for the LoadBalancer.
* Edit `kube/dnscrypt-init-job.yml` and change `example.com` to your desired hostname.
* Edit `kube/dnscrypt-srv.yml` and change `loadBalancerIP` to your static IP.
* Run `kubectl create -f kube/dnscrypt-init-job.yml` to setup your keys.
* Run `kubectl create -f kube/dnscrypt-deployment.yml` to deploy the dnscrypt server.
* Run `kubectl create -f kube/dnscrypt-srv.yml` to expose your server to the world.

To get your public key just view the logs for the `dnscrypt-init` job. The public
IP for your server is merely the `dnscrypt` service address.

Coming up next
==============

- Better isolation of the certificate signing process, in a dedicated container.
