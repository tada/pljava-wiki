![PL/Java](https://raw.github.com/tada/pljava/gh-pages/images/pljava_logo.jpg)

## 2017/06 Breaking news: workaround crash with Stack Guard-hardened kernels

As of late June 2017, Linux kernel vendors are shipping updates that harden
the kernel against certain so-called stack-smash attacks. The hardened
kernels change the mapping of memory just below the stack, and cause Java
to crash with a `SIGBUS` error (as reported elsewhere, not only in PL/Java).
If you experience this, add `-Xss2M` (or larger than 2M, if a larger stack
size is needed by your application) to `pljava.vmoptions`.

For more information, see [PL/Java issue #129][i129] and (for Red Hat
subscribers) [this Red Hat solutions document][rhsol].

[i129]: https://github.com/tada/pljava/issues/129
[rhsol]: https://access.redhat.com/solutions/3091371

# Welcome to PL/Java

If you have comments or ideas regarding this wiki, please convey them on the
[Mailing List](http://lists.pgfoundry.org/mailman/listinfo/pljava-dev).
A great deal of information can also be found at
the [project information site][phs].

[phs]: https://tada.github.io/pljava/

## Overview

PL/Java is a free add-on module that brings Java™ Stored Procedures, Triggers,
and Functions to the [PostgreSQL™](http://www.postgresql.org/) backend. The
development started late 2003 and the first release of PL/Java arrived in
January 2005. The project is released under the [[PLJava License]] license.

## Features

* Ability to write functions, triggers, user-defined types, ...
    using recent Java versions. (To see the currently-targeted versions,
    please see [the versions page][tvp].)

[tvp]: https://tada.github.io/pljava/build/versions.html

* Standardized utilities (modeled after the SQL 2003 proposal) to install and
    maintain Java code in the database.
* Standardized mappings of parameters and result. Supports scalar and
    composite user-defined types (UDTs), pseudo types, arrays, and sets.
* An embedded, high performance JDBC driver utilizing the internal PostgreSQL
    SPI routines.
* Metadata support for the JDBC driver. Both DatabaseMetaData and
    ResultSetMetaData are included.
* Integration with PostgreSQL savepoints and exception handling.
* Ability to use IN, INOUT, and OUT parameters
* Two language handlers, `javau` (functions not restricted in behavior,
    only superusers can create them) and `java` (functions run under a
    security manager blocking filesystem access, users who can create them
    configurable with `GRANT`/`REVOKE`)
* Transaction and Savepoint listeners enabling code execution when a
    transaction or savepoint is commited or rolled back.

*PL/Java earlier supported GCJ, but targets conventional Java
virtual machines for current development.*

## Documentation

The first stop for *up-to-date* documentation should be the
[project information site][phs].

You may also find useful information via the wiki links below.
Information here will be migrating to the [project information site][phs]
as it is brought up to date.

[[Installation Guide]]  
[[User Guide]]  
[[Contribution Guide]]  

## Resources

*Note: at the moment, the Downloads available below are quite old.
Please consider [Building PL/Java][bpj] using the source here on GitHub.*

[bpj]: https://tada.github.io/pljava/build/build.html
[Downloads](http://pgfoundry.org/frs/?group_id=1000038)  
[Mailing List](http://lists.pgfoundry.org/mailman/listinfo/pljava-dev)    
[Bug Tracking](/tada/pljava/issues)  
[Older bug tracker at PgFoundry](http://pgfoundry.org/tracker/?group_id=1000038)

## Technology

[[Technology in Brief]]  
[[The choice of JNI]]
