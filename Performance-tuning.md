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

[hstune]: http://tada.github.io/pljava/install/vmoptions.html

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

In PostgreSQL 11beta3, the resulting document has the following size (after
PL/Java and the example code have been loaded):

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
  xmltable.*
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

`EXPLAIN ANALYZE` reported timings in milliseconds:

iteration | `pg` | `hs` | `hs-cds` | `hs-appcds` | `j9` | `j9q` | `j9s` | `j9qs`
----------|-----:|-----:|---------:|------------:|-----:|------:|------:|-------:
1st |908.231|1888.859|1837.186|1539.781|3250.965|3095.733|2443.649|2644.991
2nd |879.483|772.545|838.082|826.558|1229.200|1855.513|1073.335|1932.083
4th |881.302|664.422|688.487|673.037|1011.018|1708.208|987.191|1912.010
8th |880.766|640.940|643.535|632.260|962.517|1660.867|952.857|1870.506
16th|880.622|654.674|682.772|627.037|967.805|1656.651|943.923|1941.888

### Discussion

* The baseline native `XMLTABLE` implementation in PostgreSQL delivers
    consistent times over successive runs. Java timings improve over successive
    early runs, as the VM identifies and reoptimizes hot areas.
* For all of the Java results, the first-iteration result includes the time
    to launch the Java virtual machine. For Hotspot, this gives a time to first
    result from 67% (best) to 108% (worst) longer than the native baseline.
* All tested Hotspot configurations are outperforming the native implementation
    as soon as the next iteration, and eventually by 22% to 28%.
* For this workload, Hotspot seems to have a striking performance advantage
    relative to OpenJ9. Possible explanations:
    * Saxon is a mature and carefully-optimized library; are its optimizations
        extremely specific to Hotspot?
    * PL/Java makes heavy use of JNI; could this pattern be less well handled
        in OpenJ9?
* OpenJ9's `-Xquickstart` is a poor fit for this workload, as it suppresses
    JIT optimization so drastically that performance improves very little on
    successive runs.
* The combination of `-Xquickstart` and `-Xshareclasses` for this workload is
    especially disappointing, probably because the two features, when combined,
    force the ahead-of-time compilation of all methods. That sounds promising,
    but not if the AOT code significantly underperforms what the optimizing JIT
    would generate.
* Memory footprint was not compared. PL/Java's [documentation][hstune] already
    has a section on plausible memory settings for Hotspot, but not for OpenJ9,
    which has a good reputation for memory frugality. Exploration would be
    worthwhile.
* There could be other workloads in which the Hotspot and OpenJ9 relative
    timings could be closer, or even reversed.
* The procedure to set up class sharing for OpenJ9 is considerably simpler
    than to set up `AppCDS` for Hotspot, enough to make OpenJ9 an attractive
    choice for workloads where the performance is more comparable.

### Variation by processor count

The results above were obtained with 6 available processor cores (12
hyperthreads). Here, the best Hotspot (`h-`) and OpenJ9 (`j-`) configurations
from above (`hs-appcds` and `j9s`, respectively) are repeated for different
numbers of cores and threads available to the backend process.

iteration | `h-4c8t` | `h-4c4t` | `h-2c4t` | `h-2c2t` | `h-1c2t` | `h-1c1t`
----------|---------:|---------:|---------:|---------:|--------:|---------:
1st |1798.020|2140.068|1871.740|2760.872|2564.169|4306.058
2nd |780.182|827.379|825.943|1054.749|1064.300|1704.112
4th |661.740|672.415|662.259|786.407|734.421|756.054
8th |619.978|653.686|641.784|659.112|678.855|722.792
16th|619.824|647.092|664.287|671.862|639.365|651.502

iteration | `j-4c8t` | `j-4c4t` | `j-2c4t` | `j-2c2t` | `j-1c2t` | `j-1c1t`
----------|---------:|---------:|---------:|---------:|--------:|---------:
1st |2413.365|2419.176|2411.006|2470.092|2457.868|3673.986
2nd |1108.991|1093.504|1050.731|1142.424|1072.635|2599.973
4th |969.465|1003.736|988.883|983.431|941.495|1032.896
8th |967.447|900.644|963.011|926.342|920.390|1032.316
16th|1113.963|932.503|1496.240|925.137|939.179|990.072

#### Discussion

Hotspot's initial startup uses parallelism to good advantage, so the startup
time suffers when cores are limited, and especially when limited to one hardware
thread. Interestingly, comparing sets with the same number of threads, in one
case independent on an equal number of cores, and in the other case hyperthreads
on half as many cores, the data above seem to favor the hyperthreaded case.
More runs might reveal whether that apparent pattern persists.

Even with only one hardware thread available, Hotspot can still produce code
that outperforms the native `libxml2` no later than the fourth iteration.

OpenJ9, while not achieving the ultimate speeds of Hotspot on this workload,
shows a first-run time that suffers less when limited to few CPUs. However,
that advantage diminishes when taking the times of the first two runs into
account.

Not shown in these tables, but as expected, the baseline PostgreSQL native
`XMLTABLE` posted timings of 893 ms first run, 877 ms second run, consistently
with the earlier values, even when limited to one core, one thread.

### Notes on methodology

#### Platform

Intel Xeon X5650 2.67 GHz, 6 cores (12 hyperthreads), 24 GB RAM, Linux.

PostgreSQL installation, Java runtimes, database, and PL/Java and Saxon
libraries and jars installed in an in-memory (`tmpfs`) filesystem.

#### Connection strings used for each test configuration

_Note: the connection strings below for the Hotspot runs with `AppCDS` contain
the option `-XX:+UnlockCommercialFeatures` because the runs were done on
Oracle Java 8 where `AppCDS` is a commercial feature, and its use in production
will need a license from Oracle. The same feature appears in OpenJDK with
Hotspot starting in Java 10, where it is not a commercial feature, and does not
require that `-XX:+UnlockCommercialFeatures` option; it is otherwise configured
in the same way._

```
\c "dbname=postgres options='-c pljava.libjvm_location=/var/tmp/nohome/jre/lib/amd64/server/libjvm.so -c pljava.vmoptions=-Djava.home=/var/tmp/nohome/jre\\\ -XX:+UseSerialGC\\\ -XX:+DisableAttachMechanism\\\ -Xshare:off -c pljava.classpath=/var/tmp/nohome/pg11/share/postgresql/pljava/pljava-1.5.1-SNAPSHOT.jar:/var/tmp/nohome/jre/lib/Saxon-HE-9.8.0-14.jar'"
\c "dbname=postgres options='-c pljava.libjvm_location=/var/tmp/nohome/jre/lib/amd64/server/libjvm.so -c pljava.vmoptions=-Djava.home=/var/tmp/nohome/jre\\\ -XX:+UseSerialGC\\\ -XX:+DisableAttachMechanism\\\ -Xshare:on -c pljava.classpath=/var/tmp/nohome/pg11/share/postgresql/pljava/pljava-1.5.1-SNAPSHOT.jar:/var/tmp/nohome/jre/lib/Saxon-HE-9.8.0-14.jar'"
\c "dbname=postgres options='-c pljava.libjvm_location=/var/tmp/nohome/jre/lib/amd64/server/libjvm.so -c pljava.vmoptions=-Djava.home=/var/tmp/nohome/jre\\\ -XX:+UseSerialGC\\\ -XX:+DisableAttachMechanism\\\ -Xshare:on\\\ -XX:+UnlockCommercialFeatures\\\ -XX:+UseAppCDS\\\ -XX:SharedArchiveFile=/var/tmp/nohome/pljava.jsa -c pljava.classpath=/var/tmp/nohome/pg11/share/postgresql/pljava/pljava-1.5.1-SNAPSHOT.jar:/var/tmp/nohome/jre/lib/Saxon-HE-9.8.0-14.jar'"
\c "dbname=postgres options='-c pljava.libjvm_location=/var/tmp/jdk8u162-b12_openj9-0.8.0/jre/lib/amd64/j9vm/libjvm.so'"
\c "dbname=postgres options='-c pljava.libjvm_location=/var/tmp/jdk8u162-b12_openj9-0.8.0/jre/lib/amd64/j9vm/libjvm.so -c pljava.vmoptions=-Xquickstart'"
\c "dbname=postgres options='-c pljava.libjvm_location=/var/tmp/jdk8u162-b12_openj9-0.8.0/jre/lib/amd64/j9vm/libjvm.so -c pljava.vmoptions=-Xshareclasses:cacheDir=/var/tmp/pljavaj9cache'"
\c "dbname=postgres options='-c pljava.libjvm_location=/var/tmp/jdk8u162-b12_openj9-0.8.0/jre/lib/amd64/j9vm/libjvm.so -c pljava.vmoptions=-Xshareclasses:cacheDir=/var/tmp/pljavaj9cache\\\ -Xquickstart'"
```

#### Jars loaded into PL/Java

The PL/Java `sqlj.install_jar` function was used to install the PL/Java
examples jar (giving it the name `ex`), with `deploy => true` to create the
function declarations, and also the `Saxon-HE-9.8.0-14.jar`, naming it `saxon`.

The PL/Java application classpath (set with `sqlj.set_classpath` on the `public`
schema), was `ex` during the Hotspot runs, and `ex:saxon` during the OpenJ9
runs. (For the Hotspot runs, the Saxon jar was placed on the system classpath
by adding it to `pljava.classpath` instead, as explained below.)

#### Setup for Hotspot

* The existing Hotspot installation on disk was copied to the `tmpfs`.
* That invalidates the paths in the supplied `classes.jsa` shared archive that
    was generated when Java was installed to its location on disk, so the
    `lib/amd64/server/classes.jsa` file was removed from the copy and
    regenerated with `java -Xshare:dump` to contain the correct paths. That
    shared archive contains only classes of the Java runtime itself.
* The shared archive for `AppCDS`, to include PL/Java implementation
    classes and the Saxon library as well as the Java runtime's classes,
    was generated in two steps:
    1. A connection string with `-XX:DumpLoadedClassList=filename` was issued
        and the test query was executed, to populate the class list with the
        needed classes.
    1. A new connection string with `-Xshare:dump` and `-XX:SharedClassListFile`
        naming the classlist file generated in the first step was issued, and
        then `SELECT sqlj.get_classpath('public');` to trigger PL/Java loading.
        Java reads the class list and generates the shared archive, and the
        backend exits.
* Because Hotspot `AppCDS` will share only classes from the system classpath,
    the `pljava.classpath` setting was altered to include
    `Saxon-HE-9.8.0-14.jar` as well as the PL/Java jar.
* Because PL/Java's security manager disallows jar loading from arbitrary
    filesystem locations, the `Saxon-HE-9.8.0-14.jar` was placed in Java's
    `jre/lib` directory and the `pljava.classpath` referred to it there.
* `AppCDS` will not share classes contained in a signed jar, and the distributed
    `Saxon-HE-9.8.0-14.jar` is signed, so the copy placed in `jre/lib` was
    "de-signed" by deleting its `TE-050AC.SF` entry and all `Name:`/`Digest:`
    sections from its `MANIFEST.MF` entry.

#### Setup for OpenJ9

* The OpenJDK with OpenJ9 download was unzipped in the `/var/tmp` `tmpfs`.
* Because PL/Java under OpenJ9 is able to share classes from the PL/Java
    application classpath (the one managed by `sqlj.set_classpath`) and not
    just the system classpath, there was no need to add the Saxon jar to
    `pljava.classpath` as there was for Hotspot. It was simply loaded
    with `sqlj.install_jar` under the name `saxon`, and put on the application
    classpath with `SELECT sqlj.set_classpath('public', 'ex:saxon');`.
* Each set of runs with sharing (`j9s`, `j9qs`) was prepared by starting a fresh
    session with the same connection string to be used for that set, and the
    `shareDir` named in that connection string empty. Sixteen runs were made
    without timing, to populate the shared cache.
* Then the same connection string was used again to start a fresh session,
    and the full set of 16 runs repeated and timed.

#### Connection strings generating `AppCDS` shared archive

_See the earlier note concerning the `-XX:+UnlockCommercialFeatures` option,
which is needed (with legal implications) to use the `AppCDS` feature in
Oracle Java. The same feature appears in OpenJDK as of Java 10, without the
need for that option or a commercial license._

```
\c "dbname=postgres options='-c pljava.libjvm_location=/var/tmp/nohome/jre/lib/amd64/server/libjvm.so -c pljava.vmoptions=-Djava.home=/var/tmp/nohome/jre\\\ -XX:+UseSerialGC\\\ -XX:+DisableAttachMechanism\\\ -Xshare:off\\\ -XX:DumpLoadedClassList=/var/tmp/nohome/pljava.classlist\\\ -XX:+UnlockCommercialFeatures\\\ -XX:+UseAppCDS -c pljava.classpath=/var/tmp/nohome/pg11/share/postgresql/pljava/pljava-1.5.1-SNAPSHOT.jar:/var/tmp/nohome/jre/lib/Saxon-HE-9.8.0-14.jar'"
\c "dbname=postgres options='-c pljava.libjvm_location=/var/tmp/nohome/jre/lib/amd64/server/libjvm.so -c pljava.vmoptions=-Djava.home=/var/tmp/nohome/jre\\\ -XX:+UseSerialGC\\\ -XX:+DisableAttachMechanism\\\ -Xshare:dump\\\ -XX:SharedClassListFile=/var/tmp/nohome/pljava.classlist\\\ -XX:+UnlockCommercialFeatures\\\ -XX:+UseAppCDS\\\ -XX:SharedArchiveFile=/var/tmp/nohome/pljava.jsa -c pljava.classpath=/var/tmp/nohome/pg11/share/postgresql/pljava/pljava-1.5.1-SNAPSHOT.jar:/var/tmp/nohome/jre/lib/Saxon-HE-9.8.0-14.jar'"
```

#### "De-signing" the Saxon jar

Hotspot's `AppCDS` will not share classes from a signed jar, so the signatures
were removed from the Saxon jar with this procedure:

```sh
zip -d Saxon-HE-9.8.0-14.jar META-INF/TE-050AC.SF
unzip Saxon-HE-9.8.0-14.jar META-INF/MANIFEST.MF
ed META-INF/MANIFEST.MF <<END-COMMANDS
/^[[:space:]]/+1,$d
wq
END-COMMANDS
zip -u Saxon-HE-9.8.0-14.jar META-INF/MANIFEST.MF
```

Stripping the signatures does not impair the operation of the open-source
Saxon-HE. It is conceivable that the commercial Saxon-PE or Saxon-EE would
object to such treatment.

#### Setup for processor-count variation

Several Linux control groups were created as follows:

```sh
mkdir /sys/fs/cgroup/cpuset/{1c1t,1c2t,2c2t,2c4t,4c4t,4c8t}
for i in /sys/fs/cgroup/cpuset/?c?t
do
  echo 0 >$i/cpuset.mems
done
echo 0       >/sys/fs/cgroup/cpuset/1c1t/cpuset.cpus
echo 0,1     >/sys/fs/cgroup/cpuset/1c2t/cpuset.cpus
echo 0,2     >/sys/fs/cgroup/cpuset/2c2t/cpuset.cpus
echo 0-3     >/sys/fs/cgroup/cpuset/2c4t/cpuset.cpus
echo 0,2,4,6 >/sys/fs/cgroup/cpuset/4c4t/cpuset.cpus
echo 0-7     >/sys/fs/cgroup/cpuset/4c8t/cpuset.cpus
```

After each new backend was established with the appropriate `\c` line,
its process ID was obtained with `SELECT pg_backend_pid();` and echoed
into `cgroup.procs` in the appropriate `cpuset` subdirectory.

The OpenJ9 class share was initially populated with one set of 16 runs
before any timing was done. Timings were then done in the order shown, from
`4c8t` to `1c1t`, and the `-Xshareclasses` option did not have `readonly`
added for the timed sets. Because OpenJ9 can continue adding JIT hints to
a class share during operation, it is possible that the later sets benefit
from JIT hints added during the earlier ones.
