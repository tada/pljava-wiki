A function or trigger in SQL resolves to a static method in a Java class. In
order for the function to execute, the appointed class must be installed in the
database. PL/Java adds a set of functions that helps installing and maintaining
the java classes. Classes are loaded into the database from normal Java
archives (AKA jars). A Jar may optionally contain a deployment descriptor that
in turn contains SQL commands to be executed when the jar is
deployed/undeployed. The functions are modeled after the standards proposed for
SQL 2003.

PL/Java implements a standardized way of passing parameters and return values.
Complex types and sets are passed using the standard JDBC ResultSet class.
Great care has been taken not to introduce any proprietary interfaces unless
absolutely necessary so that Java code written using PL/Java becomes as
database agnostic as possible.

A JDBC driver is included in PL/Java. This driver is written directly on top of
the PostgreSQL internal SPI routines. This driver is essential since it's very
common for functions and triggers to reuse the database. When they do, they
must use the same transactional boundaries that where used by the caller.

PL/Java is optimized for performance. The Java virtual machine executes within
the same process as the backend itself. This vouches for a very low call
overhead. PL/Java is designed with the objective to enable the power of Java to
the database itself so that database intensive business logic can execute as
close to the actual data as possible.

The standard Java Native Interface (JNI) is used when bridging calls from the
backend into the Java VM and vice versa. Please read the rationale behind
[[The choice of JNI]] and a more in-depth discussion about some
implementation details.

The versions of PostgreSQL and Java targeted by current PL/Java development
can be reviewed on [the versions page][tvp].

[tvp]: https://tada.github.io/pljava/build/versions.html
