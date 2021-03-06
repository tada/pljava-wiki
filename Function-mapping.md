# Functions

A Java function is declared with the name of a class and a public static method
on that class. The class will be resolved using the classpath that has been
defined for the schema where the function is declared. If no classpath has been
defined for that schema, the `public` schema is used. Please note that the
*system classloader* will take precedence always. There is no way to override
classes loaded with that loader.

The following function can be declared to access the static method
`getProperty` on the `java.lang.System` class:

```sql
CREATE FUNCTION getsysprop(VARCHAR)
  RETURNS VARCHAR
  AS 'java.lang.System.getProperty'
  LANGUAGE java;

SELECT getsysprop('java.version');
```

Both the parameters and the return value can be explicitly stated so the above
example could also have been written:

```sql
CREATE FUNCTION getsysprop(VARCHAR)
  RETURNS VARCHAR
  AS 'java.lang.String=java.lang.System.getProperty(java.lang.String)'
  LANGUAGE java;
```

This way of declaring the function is useful when the default mapping is
inadequate. PL/Java will use a standard PostgreSQL explicit cast when the SQL
type of the parameter or return value does not correspond to the Java type
defined in the mapping.

_Note: the "explicit cast" here referred to is not accomplished by creating
an actual SQL CAST expression, but by (mostly) equivalent means. At the time
of this writing, two special cases are not yet implemented._

## SQL generation

The simplest way to write the SQL function declaration that corresponds to
your Java code is to have the Java compiler do it for you:

```java
public class Hello {
  @Function
  public static String hello(String toWhom) {
    return "Hello, " + toWhom + "!";
  }
}
```

When this function is compiled, a "deployment descriptor" containing the right
SQL function declaration is also produced. When it is included in a `jar` file
with the compiled code, PL/Java's `sqlj.install_jar` function will create the
SQL function declaration at the same time it loads the jar. See the full
[hello world example](https://tada.github.io/pljava/use/hello.html) for more.
