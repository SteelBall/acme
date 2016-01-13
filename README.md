# ACME Client Utilities [![Build Status](https://travis-ci.org/hlandau/acme.svg?branch=master)](https://travis-ci.org/hlandau/acme) [![Issue Stats](http://issuestats.com/github/hlandau/acme/badge/issue)](http://issuestats.com/github/hlandau/acme)

acmetool is an easy-to-use command line tool for automatically acquiring
certificates from ACME servers (such as Let's Encrypt). Designed to flexibly
integrate into your webserver setup to enable automatic verification. Unlike
the official Let's Encrypt client, this doesn't modify your web server
configuration.

You can perform verifications using port 80 or 443 (if you don't yet have a
server running on one of them); via webroot; by configuring your webserver to
proxy requests for `/.well-known/acme-challenge/` to a special port (402) which
acmetool can listen on; or by configuring your webserver not to listen on port
80, and instead running acmetool's built in HTTPS redirector (and challenge
responder) on port 80. This is useful if all you want to do with port 80 is
redirect people to port 443.

You can run acmetool on a cron job to renew certificates automatically (`acmetool --batch`).  The
preferred certificate for a given hostname is always at
`/var/lib/acme/live/HOSTNAME/{cert,chain,fullchain,privkey}`. You can configure
acmetool to reload your webserver automatically when it renews a certificate.

acmetool is intended to be "magic-free". All of acmetool's state is stored in a
simple, comprehensible directory of flat files. [The schema for this directory
is documented.](https://github.com/hlandau/acme/blob/master/_doc/SCHEMA.md)

acmetool is intended to work like "make". The state directory expresses target
domain names, and whenever acmetool is invoked, it ensures that valid
certificates are available to meet those names. Certificates which will expire
soon are renewed. acmetool is thus idempotent and minimises the use of state.

acmetool can optionally be used [without running it as
root.](https://github.com/hlandau/acme/blob/master/_doc/NOROOT.md) If you have
existing certificates issued using the official client, acmetool can import
those certificates, keys and account keys (`acmetool import-le`).

acmetool supports both RSA and ECDSA keys and certificates. acmetool's
notification hooks system allows you to write arbitrary shell scripts to be
executed when new certificates are obtained. By default, this is used to reload
webservers automatically, but it can also be used to distribute certificates to
other servers or for other purposes.

## Getting Started

**Binary releases:** [Binary releases are available.](https://github.com/hlandau/acme/releases)

Download the release appropriate for your platform and simply copy the
`acmetool` binary to `/usr/bin`.

`_cgo` releases are preferred over non-`_cgo` releases where available, but
non-`_cgo` releases may be more compatible with older OSes.

**Ubuntu users:** A binary release PPA, `ppa:hlandau/rhea` (package `acmetool`) is available.
```bash
$ sudo add-apt-repository ppa:hlandau/rhea
$ sudo apt-get update
$ sudo apt-get install acmetool
```

You can also [download .deb files manually.](https://launchpad.net/~hlandau/+archive/ubuntu/rhea/+packages)

(Note: There is no difference between the .deb files for different Ubuntu release codenames; they are interchangeable and completely equivalent.)

**Debian users:** The Ubuntu binary release PPA also works with Debian:
```
# echo 'deb http://ppa.launchpad.net/hlandau/rhea/ubuntu xenial main' > /etc/apt/sources.list.d/rhea.list
# apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 9862409EF124EC763B84972FF5AC9651EDB58DFA
# apt-get update
# apt-get install acmetool
```

You can also [download .deb files manually.](https://launchpad.net/~hlandau/+archive/ubuntu/rhea/+packages)

(Note: There is no difference between the .deb files for different Ubuntu release codenames; they are interchangeable and completely equivalent.)

**Arch Linux users:** [An AUR PKGBUILD for building from source is available.](https://aur.archlinux.org/packages/acmetool-git/)

```bash
$ wget https://aur.archlinux.org/cgit/aur.git/snapshot/acmetool-git.tar.gz
$ tar xvf acmetool-git.tar.gz
$ cd acmetool-git
$ makepkg -s
$ sudo pacman -U ./acmetool*.pkg.tar.xz
```

**Building from source:** You will need Go installed to build from source.

If you are on Linux, you will need to make sure the development files for
`libcap` are installed. This is probably a package for your distro called
`libcap-dev` or `libcap-devel` or similar.

```bash
$ git clone https://github.com/hlandau/acme
$ cd acme
$ make && sudo make install

  # (People familiar with Go with a GOPATH setup can alternatively use go get/go install:)
  $ go get github.com/hlandau/acme/cmd/acmetool
```

### After installation

```bash
# Run the quickstart wizard. Sets up account, cronjob, etc.
$ sudo acmetool quickstart

# Configure your webserver to serve challenges if necessary.
# See https://github.com/hlandau/acme/blob/master/_doc/WSCONFIG.md
$ ...

# Request the hostnames you want:
$ sudo acmetool want example.com www.example.com

# Now you have certificates:
$ ls -l /var/lib/acme/live/example.com/
```

The `quickstart` subcommand is a recommended wizard
which guides you through the setup of ACME on your system.

The `want` subcommand states that you want a certificate for the given hostnames.
(If you want separate certificates for each of the hostnames, run the want
subcommand separately for each hostname.)

The default subcommand, `reconcile`, is like "make" and makes sure all desired
hostnames are satisfied by valid certificates which aren't soon to expire.
`want` calls `reconcile` automatically.

If you run `acmetool reconcile` on a cronjob to facilitate automatic renewal,
pass `--batch` to ensure it doesn't attempt to interact with a terminal.

You can increase logging severity for debugging purposes by passing
`--xlog.severity=debug`.

## Validation Options

<img src="https://i.imgur.com/w8TbgLL.png" align="right" alt="[screenshot]" />

**Webroot:** acmetool can place challenge files in a given directory, allowing your normal
web server to serve them. The files must be served from the path you specify at
`/.well-known/acme-challenge/`.

[Information on configuring your web server.](https://hlandau.github.io/acme/userguide#web-server-configuration)

**Proxy:** acmetool can respond to validation challenges by serving them on port 402. In
order for this to be useful, you must configure your webserver to proxy
requests under `/.well-known/acme-challenge/` to
`http://127.0.0.1:402/.well-known/acme-challenge`.

[Information on configuring your web server.](https://hlandau.github.io/acme/userguide#web-server-configuration)

**Redirector:** `acmetool redirector` starts an HTTP server on port 80 which redirects all
requests to HTTPS, as well as serving any necessary validation responses. The
`acmetool quickstart` wizard can set it up for you if you use systemd.
Otherwise, you'll need to configure your system to run `acmetool redirector
--service.uid=USERNAME --service.daemon=1` as a service, where `USERNAME` is
the username you want the daemon to drop to.

Make sure your web server is not listening on port 80.

**Listen:** If you are for some reason not running anything on port 80 or 443, acmetool
will use those ports. Either port being available is sufficient. This is only
really useful for development purposes.

## Renewal

acmetool will try to renew certificates automatically once they are 30 days
from expiry, or 66% through their validity period, whichever is lower.
Note that Let's Encrypt currently issues 90 day certificates.

acmetool will exit with an error message with nonzero exit status if it cannot
renew a certificate, so it is suitable for use in a cronjob. Ensure your system
is configured so that you get notifications of failing cronjobs.

If a cronjob fails, you should intervene manually to see what went wrong by
running `acmetool` (possibly with `--xlog.severity=debug` for verbose logging).

## Library

The [client library which these utilities use](https://github.com/hlandau/acme/tree/master/acmeapi) can be used independently by any Go code. [README and source code.](https://github.com/hlandau/acme/tree/master/acmeapi). [Godoc.](https://godoc.org/github.com/hlandau/acme/acmeapi)

## Comparison with...

**Let's Encrypt Official Client:** A heavyweight Python implementation which is
a bit too “magic” for my tastes. Tries to mutate your webserver configuration
automatically.

acmetool is a single-file binary which only depends on basic system libraries
(on Linux, these are libc, libpthread, libcap, libattr). It doesn't do anything
to your webserver; it just places certificates at a standard location and can
also reload your webserver (whichever webserver it is) by executing hook shell
scripts.

acmetool isn't based around individual transactions for obtaining certificates;
it's about satisfying expressed requirements by any means necessary. Its
comprehensible, magic-free state directory makes it as stateless and idempotent
as possible.

**lego:** Like acmetool, [xenolf/lego](https://github.com/xenolf/lego) provides
a library and client utility. The utility provides commands for creating
certificates, but doesn't provide a compelling system for managing the lifetime
of the short-lived certificates offered by Let's Encrypt. The user is expected
to generate and install all certificates manually.

**gethttpsforfree:**
[diafygi/gethttpsforfree](https://github.com/diafygi/gethttpsforfree) provides
an HTML file which uses JavaScript to make requests to an ACME server and
obtain certificates. It's a functional user interface, but like lego it
provides no answer for the automation issue, and is thus impractical given the
short lifetime of certificates issued by Let's Encrypt.

### Comparison, list of client implementations

<table>
<tr><td></td><th>acmetool</th><th><a href="https://github.com/letsencrypt/letsencrypt">letsencrypt</a></th><th><a href="https://github.com/xenolf/lego">lego</a></th><th><a href="https://github.com/diafygi/gethttpsforfree">gethttpsforfree</a></th></tr>
<tr><td>Automatic renewal</td><td>Yes</td><td>Not yet</td><td>No</td><td>No</td></tr>
<tr><td>State management</td><td>Yes†</td><td>Yes</td><td>—</td><td>—</td></tr>
<tr><td>Single-file binary</td><td>Yes</td><td>No</td><td>Yes</td><td>Yes</td></tr>
<tr><td>Quickstart wizard</td><td>Yes</td><td>Yes</td><td>No</td><td>No</td></tr>
<tr><td>Modifies webserver config</td><td>No</td><td>By default</td><td>No</td><td>No</td></tr>
<tr><td>Non-root support</td><td><a href="https://github.com/hlandau/acme/blob/master/_doc/NOROOT.md">Optional</a></td><td>Optional</td><td>Optional</td><td>—</td></tr>
<tr><td>Supports Apache</td><td>Yes</td><td>Yes</td><td>—</td><td>—</td></tr>
<tr><td>Supports nginx</td><td>Yes</td><td>Experimental</td><td>—</td><td>—</td></tr>
<tr><td>Supports HAProxy</td><td>Yes</td><td>No</td><td>—</td><td>—</td></tr>
<tr><td>Supports any web server</td><td>Yes</td><td>Webroot‡</td><td>—</td><td>—</td></tr>
<tr><td>Authorization via webroot</td><td>Yes</td><td>Yes</td><td>—</td><td>Manual</td></tr>
<tr><td>Authorization via port 80 redirector</td><td>Yes</td><td>No</td><td>No</td><td>No</td></tr>
<tr><td>Authorization via proxy</td><td>Yes</td><td>No</td><td>No</td><td>No</td></tr>
<tr><td>Authorization via listener§</td><td>Yes</td><td>Yes</td><td>Yes</td><td>No</td></tr>
<tr><td>Import state from official client</td><td>Yes</td><td>—</td><td>—</td><td>—</td></tr>
<tr><td>Windows (basic) support</td><td>No</td><td>No</td><td>Yes</td><td>—</td></tr>
<tr><td>Windows integration support</td><td>No</td><td>No</td><td>No</td><td>—</td></tr>

</table>

† acmetool has a different philosophy to state management and configuration to
the Let's Encrypt client; see the beginning of this README.

‡ The webroot method does not appear to provide any means of reloading the
webserver once the certificate has been changed, which means auto-renewal
requires manual intervention.

§ Requires downtime.

This table is maintained in good faith; I believe the above comparison to be
accurate. If notified of any inaccuracies, I will rectify the table and publish
a notice of correction here:

  - This table previously stated that the official Let's Encrypt client doesn't
    support non-root operation. This was incorrect; it can be installed at user
    level and be configured to use user-writable directories.

## Documentation & Support

For more documentation see:
- [User Guide](https://hlandau.github.io/acme/userguide)
- [Troubleshooting](https://hlandau.github.io/acme/userguide#troubleshooting)
- [FAQ](https://hlandau.github.io/acme/userguide#faq)
- [manpage](https://hlandau.github.io/acme/acmetool.8)

If your question or issue isn't resolved by any of the above, file an issue.

IRC: [#acmetool](irc://chat.freenode.net/#acmetool) on [Freenode](http://freenode.net/).

## Licence

    © 2015—2016 Hugo Landau <hlandau@devever.net>    MIT License

