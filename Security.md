## Installation

Only a PostgreSQL super user can install PL/Java. The PL/Java utility functions
are installed as "security definer" so that they execute with the access
permissions that where granted to the creator of the functions.

## Trusted vs. untrusted language

PL/Java can declare two language entries in SQL: `java` and `javau`.
Following the conventions of other PostgreSQL PLs, the 'untrusted' language
(`javau`) places no restrictions on what the Java code can do, while the
'trusted' language (`java`) installs a security manager that restricts access
to the filesystem.

`GRANT/REVOKE USAGE ON LANGUAGE java` can be used to regulate which users
are able to create functions in the `java` language. For the `javau` language,
regardless of permissions, only superusers can create functions.

## Execution of the deployment descriptor

The [install_jar](SQL Functions#wiki-install_jar),
[replace-jar](SQL Functions#wiki-replace_jar), and
[remove_jar](SQL Functions#wiki-remove_jar)
utility functions optionally execute commands found in a
[[SQL deployment descriptor]]. Such commands are executed with the
permissions of the caller. In
other words, although the utility function is declared with "security definer",
it switches back to the session user during execution of the deployment
descriptor commands.

## Classpath manipulation

The utility function [set_classpath](SQL Functions#wiki-set_classpath) requires
that the caller of the function has been granted _CREATE_ permission on the
affected schema.
