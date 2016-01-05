# Creating a scalar (or, base) user-defined type

This text assumes that you have some familiarity with how scalar types are
created and added to the PostgreSQL type system. For more info on that topic,
please read [this chapter in the PostgreSQL docs][xtypes].

[xtypes]: http://www.postgresql.org/docs/8.4/static/xtypes.html

Creating new scalar type using Java functions is very similar to how they are
created using C functions from an SQL perspective but of course very different
when looking at the actual implementation. Java stipulates that the mapping
between a Java class and a corresponding SQL type should be done using the
interfaces `java.sql.SQLData`, `java.sql.SQLInput`, and
`java.sql.SQLOutput` and that is what PL/Java is using. In addition, the
PostgreSQL type system stipulates that each type must have a textual
representation.

Let us create a type called `javatest.complex` (similar to the complex
type used in the PostgreSQL documentation). The name of the corresponding
Java class will be `org.postgresql.pljava.example.ComplexScalar`.

## The Java code for the scalar type

### Prerequisites for the Java implementation

The java class for a scalar UDT must implement the `java.sql.SQLData`
interface. In addition, it must also implement a method
`static T parse(String stringRepresentation, String typeName)` where `T` will
be the name of the class--that is, `parse` will create and return an instance
of the class--and the `java.lang.String toString()` method.
The `toString()` method must return something
that the `parse()` method can parse.

```java
package org.postgresql.pljava.example;

import java.io.IOException;
import java.io.StreamTokenizer;
import java.io.StringReader;
import java.sql.SQLData;
import java.sql.SQLException;
import java.sql.SQLInput;
import java.sql.SQLOutput;
import java.util.logging.Logger;

import org.postgresql.pljava.annotation.Function;
import org.postgresql.pljava.annotation.SQLType;
import org.postgresql.pljava.annotation.BaseUDT;

import static org.postgresql.pljava.annotation.Function.Type.IMMUTABLE;
import static
       org.postgresql.pljava.annotation.Function.OnNullInput.RETURNS_NULL;

@BaseUDT(schema="javatest", name="complex",
         internalLength=16, alignment=BaseUDT.Alignment.DOUBLE)
public class ComplexScalar implements SQLData
{
   private double m_x;
   private double m_y;
   private String m_typeName;

   @Function(effects=IMMUTABLE, onNullInput=RETURNS_NULL)
   public static ComplexScalar parse(String input, String typeName)
   throws SQLException
   {
      try
      {
         StreamTokenizer tz = new StreamTokenizer(new StringReader(input));
         if(tz.nextToken() == '('
         && tz.nextToken() == StreamTokenizer.TT_NUMBER)
         {
            double x = tz.nval;
            if(tz.nextToken() == ','
            && tz.nextToken() == StreamTokenizer.TT_NUMBER)
            {
               double y = tz.nval;
               if(tz.nextToken() == ')')
               {
                  return new ComplexScalar(x, y, typeName);
               }
            }
         }
         throw new SQLException("Unable to parse complex from string \""
	    + input + '"');
      }
      catch(IOException e)
      {
         throw new SQLException(e.getMessage());
      }
   }

   public ComplexScalar()
   {
   }

   public ComplexScalar(double x, double y, String typeName)
   {
      m_x = x;
      m_y = y;
      m_typeName = typeName;
   }

   @Override
   public String getSQLTypeName()
   {
      return m_typeName;
   }

   @Function(effects=IMMUTABLE, onNullInput=RETURNS_NULL)
   @Override
   public void readSQL(SQLInput stream, String typeName) throws SQLException
   {
      m_x = stream.readDouble();
      m_y = stream.readDouble();
      m_typeName = typeName;
   }

   @Function(effects=IMMUTABLE, onNullInput=RETURNS_NULL)
   @Override
   public void writeSQL(SQLOutput stream) throws SQLException
   {
      stream.writeDouble(m_x);
      stream.writeDouble(m_y);
   }

   @Function(effects=IMMUTABLE, onNullInput=RETURNS_NULL)
   @Override
   public String toString()
   {
      s_logger.info(m_typeName + " toString");
      StringBuffer sb = new StringBuffer();
      sb.append('(');
      sb.append(m_x);
      sb.append(',');
      sb.append(m_y);
      sb.append(')');
      return sb.toString();
   }

   /* Meaningful code that actually does something with this type was
    * intentionally left out.
    */
}
```

The class itself is annotated with `@BaseUDT`, giving its SQL schema and name,
and the length and alignment needed for its internal, stored form.

Because the compiler knows the class is a `BaseUDT`, it already expects the
`parse`, `toString`, `readSQL`, and `writeSQL` methods to be present, and
will generate the correct SQL to declare them as functions to PostgreSQL.
The `@Function` annotations are only there to declare the immutability and
on-null-input behavior for those methods, because those values are not the
defaults when declaring a function.
