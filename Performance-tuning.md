# Tuning PL/Java performance

As of 2018, there is a strong selection of Java runtimes that can be used
to back PL/Java, including at least:

* Oracle's Java (and Hotspot JVM)
* OpenJDK (with Hotspot JVM)
* OpenJDK (with Eclipse OpenJ9 JVM)

These JVMs offer a wide variety of configurable options affecting both memory
footprint and time performance of applications using PL/Java. The options
include initial and limit sizes for different memory regions, aggressiveness
of just-in-time and ahead-of-time compilation, choice of garbage-collection
algorithm, and various forms of shared-memory caching of precompiled classes.

The formal PL/Java documentation contains
[a fairly extensive treatment of useful Hotspot settings][hstune], including
a section on plausible minimum settings for memory footprint achievable with
different class-sharing and garbage-collector settings. The documentation there
of the comparable options and limits for OpenJ9 is more sparse at present.

This wiki page is intended as a clearinghouse for tuning tips and performance
measurements for various PL/Java workloads and the available Java runtimes,
that can be updated more actively between releases of the formal documentation.

## Tip for quickly comparing runtime configurations

Once the PL/Java extension is installed in a database, in any newly-created
session, the Java virtual machine is started on the first use of a PL/Java
function. The JVM that is started, and how, are determined by the settings
of `pljava.*` configuration variables in effect at that moment, most
importantly:

* `pljava.libjvm_location` selects which Java runtime will be used
* `pljava.vmoptions` supplies the options to be passed to it

Therefore, all without exiting `psql`, a new Java runtime or combination of
options can be tested by switching to a new connection with `\c`, setting those
options differently, and again calling the PL/Java function of interest.

It can be convenient to include the settings on the psql `\c` line. For example,
to time `functionOfInterest()` on two different Java runtimes:

    \c "dbname=postgres options='-c pljava.libjvm_location=/path/to/oracle/.../libjvm.so'"
    EXPLAIN ANALYZE SELECT functionOfInterest();
    \c "dbname=postgres options='-c pljava.libjvm_location=/path/to/openj9/.../libjvm.so'"
    EXPLAIN ANALYZE SELECT functionOfInterest();

For obvious reasons, the `pljava.libjvm_location` and `pljava.vmoptions`
variables require privilege to set, so the connection needs to be made
with superuser credentials.

## Sample workload: Java XML manipulation

We will create a table containing a single XML document:

```sql
CREATE TABLE catalog_as_xml AS
SELECT schema_to_xml('pg_catalog', true, false, '') AS x;
```

In PostgreSQL 11beta3, the resulting document has the following size:

```sql
SELECT octet_length(xml_send(x)) AS uncompressed, pg_column_size(x) AS toasted
FROM catalog_as_xml;
```
 uncompressed | toasted 
-------------:|--------:
14049808 | 1130828

A test query will return the string value of every element whose string value
is exactly six characters (a query that may be artificial and contrived, but
can be expressed nearly identically in XML Query (the standard-mandated language
for SQL `XMLTABLE`) and in the PostgreSQL native `XMLTABLE` syntax, which is
limited to XPath 1.0).

The baseline will be the query expressed in XPath 1.0 using the PostgreSQL
`XMLTABLE` function:

```sql
EXPLAIN ANALYZE SELECT
  xmltable.*
FROM
  catalog_as_xml,
  XMLTABLE('//*[string-length(.) = 6]'
	   PASSING x
	   COLUMNS s text PATH 'string(.)'
	  );
```

It will be compared to the equivalent query expressed in XQuery 1.0
and the `"xmltable"` function defined in the not-built-by-default
`org.postgresql.pljava.example.saxon.S9` example, relying on the Saxon-HE
library:

```sql
EXPLAIN ANALYZE SELECT
  "xmltable".*
FROM
  catalog_as_xml,
  LATERAL (SELECT x AS ".") AS p,
  "xmltable"('//*[string-length(.) eq 6]',
	     PASSING => p,
	     COLUMNS => array[ 'string(.)' ]
	    )		  AS (  s text     );
```

The Java query will be run in both Oracle Java 8 (on the Hotspot JVM) and
OpenJDK 8 (with the OpenJ9 JVM), with different choices of class-sharing
options:

 tag | description
----:|-------------
`pg`|Baseline, PostgreSQL `XMLTABLE`
`hs`|Hotspot, no sharing
`hs-cds`|Hotspot, class data sharing (Java runtime classes only)
`hs-appcds`|Hotspot, AppCDS (commercial feature), Java runtime, PL/Java, Saxon
`j9`|OpenJ9, no `-Xquickstart`, no sharing
`j9q`|OpenJ9, `-Xquickstart`, no sharing
`j9s`|OpenJ9, no `-Xquickstart`, sharing (Java runtime, PL/Java, Saxon)
`j9qs`|OpenJ9, `-Xquickstart`, sharing (as above)

iteration | `hs` | `hs-cds` | `hs-appcds` | `j9` | `j9q` | `j9s` | `j9qs`
----------|-----:|---------:|------------:|-----:|------:|------:|-------:
1st |908.231|1969.658|1889.124|1544.612|3250.965|3095.733|2443.649|2644.991
2nd |879.483|747.658|777.894|719.146|1229.200|1855.513|1073.335|1932.083
4th |881.302|694.945|683.854|665.100|1011.018|1708.208|987.191|1912.010
8th |880.766|649.411|640.055|617.055|962.517|1660.867|952.857|1870.506
16th|880.622|679.538|675.594|655.664|967.805|1656.651|943.923|1941.888
