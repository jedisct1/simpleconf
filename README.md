SimpleConf
==========

A deceptively simple way to add a configuration files to an existing
command-line application.

1. Define a `configuration file property -> command-line option` map
2. Include a single C file
3. Done.

The grammar is defined at runtime, so that application plugins can
register whatever they want.

SimpleConf is the configuration file parser used in
[pure-ftpd](https://github.com/jedisct1/pure-ftpd) and
[dnscrypt-proxy](https://github.com/jedisct1/dnscrypt-proxy).

It is a trivial piece of code, but it is very generic and can easily be reused.
Hence this standalone-version, in case it could be useful to other
projects.

--(more documentation to come. In the meantime, you may want take a look at
what a map definition looks like in
[pure-ftpd](https://github.com/jedisct1/dnscrypt-proxy/blob/master/src/proxy/simpleconf_dnscrypt.h)
and
[dnscrypt-proxy](https://github.com/jedisct1/pure-ftpd/blob/master/src/simpleconf_ftpd.h)
)--
