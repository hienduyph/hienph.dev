---
title: "How to run Kerio VPN Client in any Linux system"
date: 2021-12-09T16:08:46+07:00
tags: ["linux", "docker", "fedora", "keriovpn", "arch"]
---

The Kerio VPN Client only has an official client for Debian-based like Ubuntu. We could use [Alien](https://wiki.debian.org/Alien) to convert DEB to RPM format and then install as usual.

But there's a warning about conflict packages, and if we force install, we could face trouble because of incompatibility.

Can we do this better?

> CONTAINERs is the rescue!

I provide full Dockerfile and related scripts at https://github.com/hienduyph/docker-keriovpn.

## Let's try to run a client

The default port is: 4090

Obtain the fingerprint

The default port is: 4090

Obtain the fingerprint

```bash
openssl s_client -connect YOUR_SERVER:YOUR_PORT < /dev/null 2>/dev/null | openssl x509 -fingerprint -md5 -noout -in /dev/stdin
```

*kerio-svc.conf*

```
<config>
  <connections>
    <connection type="persistent">
      <server>YOUR_SERVER</server>
      <port>YOUR_PORT</port>
      <username>YOUR_USERNAME</username>
      <password>YOUR_PASSWORD</password>
      <fingerprint>FINGERPRINT_ABOVE</fingerprint>
      <active>1</active>
    </connection>
  </connections>
</config>
```

**Spin up the client**
```bash
docker run -d --name keriovpn --net=host --privileged -v /path/to/kerio-kvc.conf:/etc/kerio-kvc.conf quay.io/hienduyph/keriovpn-client
```

From now, you could check the container logs to see detail the login address, and ask your Network Team about DNS server.

Hope this help!
