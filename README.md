[![Build Status](https://travis-ci.org/bluejekyll/trust-dns.svg?branch=master)](https://travis-ci.org/bluejekyll/trust-dns)
[![Coverage Status](https://coveralls.io/repos/github/bluejekyll/trust-dns/badge.svg?branch=master)](https://coveralls.io/github/bluejekyll/trust-dns?branch=master)
[![](http://meritbadge.herokuapp.com/trust-dns)](https://crates.io/crates/trust-dns)


# trust-dns

A Rust based DNS client and server, built to be safe and secure from the
ground up.

[API documentation](http://trust-dns.org/target/doc/trust_dns/index.html)

# Goals

- Build a safe and secure DNS server and client with modern features.
- No panics, all code is guarded
- Use only safe Rust, and avoid all panics with proper Error handling
- Use only stable Rust
- Protect against DDOS attacks (to a degree)
- Support options for Global Load Balancing functions
- Make it dead simple to operate

# Status:

## Client

Using the client is safe. The client is currently hardcoded to a 5 second,
 timeout. I'll make this configurable if people ask for that, please file a
 request for any features. Please send feedback! It currently does not cache
 responses, if this is a feature you'd like earlier rather than later, post a
 request. The validation of DNSSec is complete including NSEC. As of now NSEC3
 is broken, and I may never plan to support it. I have some alternative ideas
 for private data in the zone.

### Unique client side implementations

These are not unique to this client, but are high level functions that hide
the details in DNS from the caller

* secure_query - DNSSec validation
* create - atomic create of a record, with authenticated request
* append - verify existence of a record and append to it
* compare_and_swap - atomic (depends on server) compare and swap
* delete_by_rdata - delete a specific record
* delete_rrset - delete an entire record set
* delete_all - delete all records sets with a given name

## Server

The server code is complete, the daemon supports IPv4 and IPv6, UDP and TCP.
 There currently is no way to limit TCP and AXFR operations, so it is still not
 recommended to put into production as TCP can be used to DOS the service.
 Master file parsing is complete and supported. There is currently no forking
 option, and the server is not yet threaded. There is still a lot of work to do
 before a server can be trusted with this externally. Running it behind a firewall
 on a private network would be safe.

Zone signing support is complete, to insert a key store a pem encoded rsa file
 in the same directory as the initial zone file with the `.key` suffix. *Note*:
 this must be only readable by the current user. If one is not present one will
 be created and written to the correct location. This also acts as the initial
 key for dynamic update SIG(0) validation. To get the public key, the `DNSKEY`
 record for the zone can be queried. This is needed to provide to other
 upstream servers to create the `DS` key. Dynamic DNS is also complete,
 if enabled, a journal file will be stored next to the zone file with the
 `jrnl` suffix. *Note*: if the key is changed or updated, it is currently the
 operators responsibility to remove the only public key from the zone, this
 allows for the `DNSKEY` to exist for some unspecified period of time during
 key rotation. Rotating the key currently is not available online and requires
 a restart of the server process.

## DNSSec status

Currently the root key is hardcoded into the system. This gives validation of
 DNSKEY and DS records back to the root. NSEC is implemented, but not NSEC3.
 Because caching is not yet enabled, it has been noticed that some DNS servers
 appear to rate limit the connections, validating RRSIG records back to the root
 can require a significant number of additional queries for those records.

Zones will be automatically resigned on any record updates via dynamic DNS.

## RFC's implemented

### Basic operations
- [RFC 1035](https://tools.ietf.org/html/rfc1035): Base DNS spec (partial, caching not yet supported)
- [RFC 3596](https://tools.ietf.org/html/rfc3596): IPv6
- [RFC 2782](https://tools.ietf.org/html/rfc2782): Service location
- [RFC 6891](https://tools.ietf.org/html/rfc6891): Extension Mechanisms for DNS

### Update operations
- [RFC 2136](https://tools.ietf.org/html/rfc2136): Dynamic Update

### Secure DNS operations
- [RFC 3007](https://tools.ietf.org/html/rfc3007): Secure Dynamic Update
- [RFC 4034](https://tools.ietf.org/html/rfc4034): DNSSEC Resource Records
- [RFC 4035](https://tools.ietf.org/html/rfc4035): Protocol Modifications for DNSSEC
- [RFC 4509](https://tools.ietf.org/html/rfc4509): SHA-256 in DNSSEC Delegation Signer
- [RFC 5702](https://tools.ietf.org/html/rfc5702): SHA-2 Algorithms with RSA in DNSKEY and RRSIG for DNSSEC
- [RFC 6840](https://tools.ietf.org/html/rfc6840): Clarifications and Implementation Notes for DNSSEC
- [RFC 6944](https://tools.ietf.org/html/rfc6944): DNSKEY Algorithm Implementation Status

## RFC's in progress or not yet implemented

### Basic operations
- [RFC 2308](https://tools.ietf.org/html/rfc2308): Negative Caching of DNS Queries
- [RFC 2317](https://tools.ietf.org/html/rfc2317): Classless IN-ADDR.ARPA delegation

### Update operations
- [RFC 1995](https://tools.ietf.org/html/rfc1995): Incremental Zone Transfer
- [RFC 1996](https://tools.ietf.org/html/rfc1996): Notify slaves of update
- [Update Leases](https://tools.ietf.org/html/draft-sekar-dns-ul-01): Dynamic DNS Update Leases
- [Long-Lived Queries](http://tools.ietf.org/html/draft-sekar-dns-llq-01): Notify with bells

### Secure DNS operations
- [RFC 5155](https://tools.ietf.org/html/rfc5155): DNSSEC Hashed Authenticated Denial of Existence
- [RFC 6975](https://tools.ietf.org/html/rfc6975): Signaling Cryptographic Algorithm Understanding
- [DNSCrypt](https://dnscrypt.org): Trusted DNS queries
- [S/MIME](https://tools.ietf.org/html/draft-ietf-dane-smime-09): Domain Names For S/MIME

# Usage

This assumes that you have [Rust](https://www.rust-lang.org) stable installed. These
presume that the trust-dns repos have already been synced to the local system:

    $ git clone https://github.com/bluejekyll/trust-dns.git
    $ cd trust-dns

## Prerequisites

-   openssl development libraries 
-   sqlite3 development libraries

    Mac OS X: using homebrew
    
        $ brew install openssl
        $ brew link --force openssl
        $ brew install sqlite
    
    Debian-based (includes Ubuntu & Raspbian): using apt-get
    
        $ apt-get install openssl
        $ apt-get install libssl-dev
        $ apt-get install libsqlite3-dev


## Testing

-   Unit tests

    These are good for running on local systems. They will create sockets for
    local tests, but will not attempt to access remote systems.

        $ cargo test

-   Functional tests

    These will try to use some local system tools for compatibility testing,
    and also make some remote requests to verify compatibility with other DNS
    systems. These can not currently be run on Travis for example.

        $ cargo test -- --ignored

-   Benchmarks

    Waiting on benchmarks to stabilize in mainline Rust.

## Building

-   Production build

        $ cargo build --release

## Running

Warning: Trust-DNS is still under development, running in production is not
recommended. The server is currently only single-threaded, it is non-blocking
so this should allow it to work with most internal loads.

-   Verify the version

        $ target/release/named --version

-   Get help

        $ target/release/named --help

# FAQ

-   Why are you building another DNS server?

    Because of all the security advisories out there for BIND.
Using Rust semantics it should be possible to develop a high performance and
safe DNS Server that is more resilient to attacks.

## License

Licensed under either of

 * Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.
