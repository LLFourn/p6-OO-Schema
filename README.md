# OO::Schema

Declare your class relationships separate from their implementation so
you can talk about them without loading them.

## Synopsis

```perl6
# Userland.pm6
# example: declaring the relationships between different OS userlands
use OO::Schema;

schema Userland {

    node Windows {
       node XP { }
    }

    node POSIX {

        node FreeBSD { }
        node OpenBSD { }

        node GNU {
            node Debian {
                node Ubuntu { }
            }
            node RHEL is path {
                node Fedora { }
                node CentOS { }
            }
        }
    }
}
```

```perl6
# then in
# Userland/Ubuntu.pm ( and Userland::Debian, GNU etc )
use Userland :node;

# Userland::Ubuntu will now magically find itself on the schema
# tree and inherit from Userland::Debian (which will do the same)
unit class Userland::Ubuntu is schema-node;

```
and finally in
```perl6

# main.pl

use Userland;
# That's enough - I don't have to load Userland::RHEL or Userland::Debian

proto install-package(Userland:D,Str:D $pkg) {*};

# but I can still talk about things inheriting from them
# with these magical short names like 'Debian' and 'RHEL'
multi install-package(Debian $userland,$pkg) {
    run 'apt-get', 'install', $pkg;
}

multi install-package(RHEL $userland,$pkg) {
    run 'yum', 'install', $pkg;
}

# now at runtime I load particular real class and make
# an instance. It just works!
my $ubuntu = (require Userland::Ubuntu).new;
my $fedora = (require Userland::Fedora).new;

install-package($ubuntu,'ntp');
install-package($fedora,'ntp');

#etc
```

## Description

**warning** this is module is experimental and subject to change

OO::Schema is for those instances when you want to have symbols
representing certain classes without loading them. Typically when you
want to use a class as a type constraint you have to load it. For example:

``` perl6
need Userland::Ubuntu;
need Userland::Debian;
need Userland::RHEL;
need Userland::Fedora;

multi do-something(Userland::Fedora:D $ul) { ... }
multi do-something(Userland::RHEL:D $ul)   { ... }
...
```

also to inherit from something you need to load it

``` perl6
need Userland::Debain;
unit class Userland::Ubuntu is Userland::Debain
```

The point of OO::Schema is provide an alternative way to do
it. Instead of declaring your class relationships within each
compunit, you declare them in a central *schema* module. You then
`use` the schema module, which exports a bunch of short-named
symbols. The symbols will reflect the same inheritance relationships

If they want to call methods on the real class you still have to load it.

### How is this achieved

When the schema module is loaded the relationship of the nodes looks like:

```perl6
use Userland;
```

```
             Userland
               /
      .---->POSIX
     /       /
   RHEL   Debian
   /       /
Fedora  Ubuntu

```

When a node-backing class is loaded marking itself with `is
schema-node`, it attaches itself to the node class and add the
recursively loads it's dependencies. Afterwards the inheritance tree
will look like:

```perl6
use Userland;
use Userland::Ubuntu;
```

```
             Userland
               /
      .---->POSIX <-- Userland::POSIX
     /       /              /
   RHEL   Debian <-- Userland::Debian
   /       /              /
Fedora  Ubuntu  <-- Userland::Ubuntu

```
