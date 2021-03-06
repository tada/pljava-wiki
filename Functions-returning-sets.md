# Set-returning functions

Returning sets is tricky. You don't want to first build a set and then return
it, since large sets would eat excessive resources. It's better to produce
one row at a time. Incidentally, that's exactly what the PostgreSQL backend
expects a function that `RETURNS SETOF <type>` to do. The `<type>` can be a
_scalar type_ such as an _int_, _float_ or _varchar_, it can be a
_complex type_, or a _RECORD_.

### Returning a SETOF &lt;scalar type&gt;

In order to return a set of a scalar type, you need create a Java method that
returns an implementation the `java.util.Iterator` interface.

```sql
CREATE FUNCTION javatest.getNames()
  RETURNS SETOF varchar
  AS 'foo.fee.Bar.getNames'
  IMMUTABLE LANGUAGE java;
```
The corresponding Java class:
```java
package foo.fee;
import java.util.Iterator;

import org.postgresql.pljava.annotation.Function;
import static org.postgresql.pljava.annotation.Function.Effects.IMMUTABLE;

public class Bar
{
    @Function(schema="javatest", effects=IMMUTABLE)
    public static Iterator<String> getNames()
    {
        ArrayList<String> names = new ArrayList<>();
        names.add("Lisa");
        names.add("Bob");
        names.add("Bill");
        names.add("Sally");
        return names.iterator();
    }
}
```

### Returning a SETOF &lt;complex type&gt;

A method returning a `SETOF <complex type>` must use either the interface
`org.postgresql.pljava.ResultSetProvider` or
`org.postgresql.pljava.ResultSetHandle`. The reason for having two interfaces
is that they cater for optimal handling of two distinct use cases. The former
is great when you want to dynamically create each row that is to be returned
from the `SETOF` function. The latter makes sense when you want to return the
result of an executed query.

### Using the ResultSetProvider interface

This interface has two methods. The
`boolean assignRowValues(java.sql.ResultSet tupleBuilder, int rowNumber)`
and the `void close()` method. The PostgreSQL query evaluator will call the
`assignRowValues()` repeatedly until it returns false or until the evaluator
decides that it does not need any more rows. It will then call `close()`.

You can use this interface the following way:
```sql
CREATE FUNCTION javatest.listComplexTests(int, int)
  RETURNS SETOF complexTest
  AS 'foo.fee.Fum.listComplexTest'
  IMMUTABLE LANGUAGE java;
```

The function maps to a static java method that returns an instance that
implements the `ResultSetProvider` interface.

```java
public class Fum implements ResultSetProvider
{
  private final int m_base;
  private final int m_increment;
  public Fum(int base, int increment)
  {
    m_base = base;
    m_increment = increment;
  }
  public boolean assignRowValues(ResultSet receiver, int currentRow)
  throws SQLException
  {
    // Stop when we reach 12 rows.
    //
    if(currentRow >= 12)
      return false;
    receiver.updateInt(1, m_base);
    receiver.updateInt(2, m_base + m_increment * currentRow);
    receiver.updateTimestamp(3, new Timestamp(System.currentTimeMillis()));
    return true;
  }
  public void close()
  {
  	// Nothing needed in this example
  }
  @Function(effects=IMMUTABLE, schema="javatest", type="complexTest")
  public static ResultSetProvider listComplexTests(int base, int increment)
  throws SQLException
  {
    return new Fum(base, increment);
  }
}
```
The `listComplexTests(int base, int increment)` method is called once. It may
return `null` if no results are available, or an instance of the
`ResultSetProvider`. Here the `Fum` class implements this interface so it
returns an instance of itself. The method
`assignRowValues(ResultSet receiver, int currentRow)`
will then be called repeatedly until it returns `false`. At that
time, `close()` will be called.

The `currentRow` parameter can be a convenience in some cases, and
unnecessary in others. It will be passed as zero on the first call,
and incremented by one on each subsequent call. If the `ResultSetProvider`
is returning results from some source (like an `Iterator`) that remembers its
own position, it can simply ignore `currentRow`.

### Using the ResultSetHandle interface

This interface is similar to the `ResultSetProvider` interface in that it has a
`close()` method that will be called at the end. But instead of having
the evaluator call a method that builds one row at a time, this method has a
method that returns a `ResultSet`. The query evaluator will iterate over
this set and deliver its contents, one tuple at a time, to the caller until a
call to `next()` returns `false` or the evaluator decides that no more
rows are needed.

Here is an example that executes a query using a statement that it obtained
using the default connection. The SQL looks like this:

```sql
CREATE FUNCTION javatest.listSupers()
  RETURNS SETOF pg_user
  AS 'org.postgresql.pljava.example.Users.listSupers'
  LANGUAGE java;

CREATE FUNCTION javatest.listNonSupers()
  RETURNS SETOF pg_user
  AS 'org.postgresql.pljava.example.Users.listNonSupers'
  LANGUAGE java;
```
And here is the Java code:
```java
public class Users implements ResultSetHandle
{
  private final String m_filter;
  private Statement m_statement;

  public Users(String filter)
  {
    m_filter = filter;
  }

  public ResultSet getResultSet()
  throws SQLException
  {
    m_statement = DriverManager.getConnection("jdbc:default:connection")
      .createStatement();
    return m_statement.executeQuery("SELECT * FROM pg_user WHERE " + m_filter);
  }

  public void close()
  throws SQLException
  {
    m_statement.close();
  }

  @Function(schema="javatest", type="pg_user")
  public static ResultSetHandle listSupers()
  {
    return new Users("usesuper = true");
  }

  @Function(schema="javatest", type="pg_user")
  public static ResultSetHandle listNonSupers()
  {
    return new Users("usesuper = false");
  }
}
```
