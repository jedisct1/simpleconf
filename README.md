SimpleConf
==========

A deceptively simple way to add a configuration files to an existing
command-line application.

1. Define a `configuration file property -> command-line option` map
2. Include a single C file
3. Done.

The grammar is defined at runtime, so that application plugins can
register whatever they want in addition to a base set of keywords.

SimpleConf is the configuration file parser used in
[pure-ftpd](https://github.com/jedisct1/pure-ftpd) and
[dnscrypt-proxy](https://github.com/jedisct1/dnscrypt-proxy).

It is a trivial piece of code, but it is very generic and can easily be reused.
Hence this standalone-version, in case it could be useful to other
projects.

Quick start
===========

The code below uses `getopt_long()` to parse command-line arguments, but SimpleConf
is compatible with pretty much anything, as it always builds a new arguments list.

```c
#include <getopt.h>
#include <stdio.h>

static struct option getopt_long_options[] = {
    { "name", 1, NULL, 'n' }, { "bell", 0, NULL, 'b' }, { NULL, 0, NULL, 0 }
};
static const char *getopt_options = "n:b";

void parse_options(int argc, char *argv[])
{
    int flag, index = 0;
    while ((flag = getopt_long(argc, argv, getopt_options,
                               getopt_long_options, &index)) != -1) {
        switch (flag) {
            case 'n': printf("Hello %s!\n", optarg); break;
            case 'b': putchar(7); break;
        }
    }
}

int main(int argc, char *argv[])
{
    parse_options(argc, argv);
    return 0;
}
```

Simple changes to the `main()` function in order to implement support for
configuration files:

```c
#include "simpleconf.h"

int main(int argc, char *argv[])
{
    static const SimpleConfEntry sc_defs[] = {
        { "Name (<any*>)", "--name=$0" },
        { "Bell? <bool>",  "--bell"}
    };
    sc_build_command_line_from_file("hello.conf", NULL, sc_defs, 2,
                                    argv[0], &argc, &argv);
    parse_options(argc, argv);
    return 0;
}
```

Done.

Our `hello` application can now load a `hello.conf` configuration file such as
the following:

```
#################################################
##                                             ##
##     Sample configuration file for Hello     ##
##                                             ##
#################################################

# Change this to your name
Name Johnny Doe

# Change to "yes" for a (visual) bell effect
Bell no
```

The parser is quite tolerant and the following configuration files would
work equally well:

```
name = Johnny Doe
bell = off
```

```
name: Johnny Doe
bell: false
```

Parsing rules
=============

The grammar is defined by a vector of `SimpleConfEntry` values. Each value maps
a property name and a pattern to a command-line option.

A pattern can be made of arbitrary characters, including white spaces:

```c
{ "PreferredFruit banana",     "--banana" },
{ "PreferredFruit kiwi fruit", "--kiwi" }
```

But once again, the parser is very tolerant, and will gladly accept any number of
spaces (white spaces and/or tabs) between `kiwi` and `fruit` in the configuration
file.

Note that the same property name can appear multiple times: the first one that
fully matches the given pattern is the one that will be translated to a command-line
option.

Patterns can include character classes:

* `<alpha>`: matches one or more alphabetic characters
* `<alnum>`: matches one of more alphanumberic characters
* `<digits>`: matches one or more digits
* `<xdigits>`: matches one or more hex digits
* `<nospace>`: matches one or more characters that are not whitespaces
* `<any>`: matches everything except whitespaces, as well as quoted strings that can include whitespaces. Given the `kiwi fruit` input, this would only match `kiwi`. Given the `"kiwi fruit"` input, this would match `kiwi fruit`.
* `<any*>`: matches everything until the end of the line, including whitespaces, without requiring quotes.
* `<bool>`: matches `yes`, `on`, `true`, `1`, `no`, `off`, `false` and `0`.

Capturing groups
----------------

The command-line translation of a config file pattern can copy parts or all
of the matching input:

```c
{ "PreferredFruit (<alpha>)", "--fruit=$0" }
```

Capture groups are delimited by round brackets, and can include anything,
although they are mostly useful with character classes:

```c
{ "WidthAndHeight (<digits>x<digits>)", "--size=$0" }
```

Up to 10 capture groups can be present in a pattern. The first one can be
referred to as `$0`, the second one as `$1` and so on until `$9`.

```c
{ "Size width:(<digits>) height:(<digits>)", "--size=$0x$1" }
```

The whole input can also be referred to as `$*`.

Boolean mapping
---------------

The addition of a command-line switch can depend on the value of a boolean
condition. In order to do so, the property name should be suffixed with the
`?` character.

```c
{ "EnableCrazyCoolFeature? <bool>", "--crazy-cool-feature" }
```

In this example, the `--crazy-cool-feature` switch will only be added if the
value of the `EnableCrazyCoolFeature` property is either `yes`, `on`, `true`
or `1`.

Basic API
---------

```c
int sc_build_command_line_from_file(const char *file_name,
                                    const SimpleConfConfig *config,
                                    const SimpleConfEntry entries[],
                                    size_t entries_count, char *app_name,
                                    int *argc_p, char ***argv_p);
```

Builds a new list of command-line arguments by parsing the configuration file
`file_name` according to the list of patterns `entries` of size `entries_count`,
and puts the result into `argv_p` and its size into `argc_p`.

`app_name` is put into `(*argv_p)[0]`, and can be the original value for `argv[0]`.

`config` can be `NULL`, or set to a `SimpleConfConfig` when using special handlers
(see below).

```c
void sc_argv_free(int argc, char *argv[]);
```

Deallocates the memory allocated by `sc_build_command_line_from_file()`.

Special handlers
----------------

Specific keywords can cause a hook being called instead of the default behavior.

This can be useful to handle options in the configuration file that have no
command-line equivalent, as well as to support recursive configuration files.

Property names starting with a `!` character trigger that behavior:

```c
{ "!Include <any*>", "$*" }
```

In this example, instead of extending the list of command-line options, the
`Include` keyword will cause a special handler to be called.

```c
SimpleConfSpecialHandlerResult special_handler(void **output_p, const char *arg, void *user_data)
{
    // ... do some custom processing using the value `arg` ...
    return SC_SPECIAL_HANDLER_RESULT_NEXT;
}
```

Possible return values are:
* `SC_SPECIAL_HANDLER_RESULT_NEXT`: keep parsing the current file
* `SC_SPECIAL_HANDLER_RESULT_ERROR`: return an error and stop parsing the
current file
* `SC_SPECIAL_HANDLER_RESULT_INCLUDE`: parse the content of the file whose name
was stored in `*output_p`, and continue parsing the current file afterwards.

`SC_SPECIAL_HANDLER_RESULT_INCLUDE` is useful to allow users to recursively
include configuration files in configuration files. A handler can sanitize
file names, turn relative file names into absolute ones, and enforce rules to
only allow specific files.

The bare minimum is to return a copy of the original file name:

```c
SimpleConfSpecialHandlerResult special_handler(void **output_p, const char *arg, void *user_data)
{
    if ((*output_p = strdup(arg)) == NULL) {
        return SC_SPECIAL_HANDLER_RESULT_ERROR;
    }
    return SC_SPECIAL_HANDLER_RESULT_INCLUDE;
}
```

The handler should be specified via a `SimpleConfConfig` value when calling
`sc_build_command_line_from_file()`. The `user_data` field can store an
arbitrary pointer, that will be conveniently available in the handler.

Real-world examples
-------------------

* [dnscrypt-proxy](https://github.com/jedisct1/dnscrypt-proxy):
[SimpleConf definition](https://github.com/jedisct1/pure-ftpd/blob/master/src/simpleconf_ftpd.h) and
[example configuration file](https://github.com/jedisct1/dnscrypt-proxy/blob/master/dnscrypt-proxy.conf)
* [pure-ftpd](https://github.com/jedisct1/pure-ftpd):
[SimpleConf definition](https://github.com/jedisct1/dnscrypt-proxy/blob/master/src/proxy/simpleconf_dnscrypt.h) and
[example configuration file](https://github.com/jedisct1/pure-ftpd/blob/master/pure-ftpd.conf.in)
