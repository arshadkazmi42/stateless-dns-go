stateless-dns-go
================
<img src="https://tools.taskcluster.net/lib/assets/taskcluster-120.png" />

A go (golang) port of https://github.com/taskcluster/stateless-dns-server/blob/master/index.js

[![GoDoc](https://godoc.org/github.com/taskcluster/stateless-dns-go?status.svg)](https://godoc.org/github.com/taskcluster/stateless-dns-go)
[![Build Status](https://travis-ci.org/taskcluster/stateless-dns-go.svg?branch=master)](http://travis-ci.org/taskcluster/stateless-dns-go)
[![Coverage Status](https://coveralls.io/repos/taskcluster/stateless-dns-go/badge.svg?branch=master&service=github)](https://coveralls.io/github/taskcluster/stateless-dns-go?branch=master)
[![License](https://img.shields.io/badge/license-MPL%202.0-orange.svg)](https://github.com/taskcluster/stateless-dns-go/blob/master/LICENSE)

stateless-dns-go provides:

* A go (golang) package for generating client dns names that can be used in
  conjunction with https://github.com/taskcluster/stateless-dns-server, that
  can be directly imported in go applications
* A command line utility to expose the features of the library, which can be
  called from any language

Domains generated by this library encode an IP-address, expiration date, a
random salt and an HMAC-SHA256 signature truncated to 128 bits.

This provides a mechanism to assign temporary sub-domains names to nodes with a
public IP-address. The same problem can also be solved with dynamic DNS server,
but such entries often requires clean-up. The beauty of this approach is that
the DNS server is state-less, so there is no stale DNS records to discard.

In TaskCluster this is used to assign temporary sub-domain names to EC2 spot
nodes, such that we can host HTTPS resources, such as live logs, without
updating and cleaning up the state of the DNS server.

Notice, that with IP-address, expiration date, random salt and HMAC-SHA256
signature encoded in the sub-domain label, you cannot decide which sub-domain
label you wish to have. Hence, this is only useful in cases were the hostname
for your node is transmitted to clients by other means, for example in a
message over RabbitMQ or as temporary entry in a database. Further more, to
serve HTTPS content you'll need a wild-card SSL certificate, for domain managed
by this DNS server.

Note, this obviously doesn't have many applications, as the sub-domain label is
stateful. It's mostly for serving HTTPS content from nodes that come and go
quickly with minimal setup, where the hostname is transmitted by other means.
Generally, any case where you might consider using the default EC2 hostname.

Sub-domain Label Generation
---------------------------
The sub-domain label encodes the following parameters:
 * `ip`, address to which the `A` record returned should point,
 * `expires`, expiration of sub-domain as number of ms since epoch,
 * `salt`, random salt, allowing for generation of multiple sub-domain labels
   for each IP-address, and,
 * `signature`, HMAC-SHA256 signature of `ip`, `expires` and `salt` truncated
   to 128 bit.

The `expires` property is encoded as a big-endian 64 bit signed integer. The
`salt` property is encoded as bit-endian 16 bit unsigned integer. All
properties are concatenated and base32 (RFC 3548) encoded to form the
sub-domain label.


Go library usage
----------------

Here is a trivial example:

```go
import (
	"fmt"
	"net"
	"time"

	"github.com/taskcluster/stateless-dns-go/hostname"
)

func PrintHostname() {
	ip := net.IPv4(byte(203), byte(43), byte(55), byte(2))
	subdomain := "foo.com"
	expires := time.Now().Add(15 * time.Minute)
	secret := "turnip-4tea-2nite"
	name := hostname.New(ip, subdomain, expires, secret)
	fmt.Printf("Hostname: '%s'\n", name)
}
```

Installing command line tool
----------------------------

__Binary installation__

Download the create-hostname command line tool for your platform from
[here](https://github.com/taskcluster/stateless-dns-go/releases).

__Source installation__

Requirements:

  * go (golang) v1.4 or higher
  * `${GOPATH}/bin` is in your path

Run:

```
go get github.com/taskcluster/stateless-dns-go/create-hostname
```

Command line usage
------------------

Creating hostnames...

```
Usage:
  create-hostname --ip IP --subdomain SUBDOMAIN --expires EXPIRES --secret SECRET
  create-hostname -h|--help
  create-hostname --version

Exit Codes:
   0: Success
   1: Unrecognised command line options
  64: Invalid IP given
  65: IP given was an IPv6 IP (IP should be an IPv4 IP)
  66: Invalid SUBDOMAIN given
  67: Invalid EXPIRES given
  68: Invalid SECRET given

Examples:
  $ create-hostname --ip 203.115.35.2 --subdomain foo.com --expires 2016-06-04T16:04:03.739Z --secret 'cheese monkey'
  znzsgaqaau2hl7h35f4owqn25s76j4h7apm3fe4qpy6pfxjk.foo.com
```

Decoding hostnames...

```
Usage:
  decode-hostname --fqdn FQDN --subdomain SUBDOMAIN --secret SECRET
  decode-hostname --help|-h
  decode-hostname --version

Exit Codes:
   0: Success
   1: Unrecognised command line options

Example:
  $ decode-hostname --fqdn aebagbaaaaadqfbf6nanb2v3zyzdeq27biltfievlqaktog2.foo.com --subdomain foo.com --secret 'Happy Birthday Pete!'
  2017/01/10 18:50:08 IP: 1.2.3.4
  2017/01/10 18:50:08 Expires: 1977-08-19 16:30:00 +0000 UTC
  2017/01/10 18:50:08 Salt: [2]uint8{0xd0, 0xea}
```

License
-------
The stateless-dns-go library is released on the MPL 2.0 license, see file
`LICENSE` for the complete license.

Testing
-------

```bash
go test ./...
```

TODO: write tests!
