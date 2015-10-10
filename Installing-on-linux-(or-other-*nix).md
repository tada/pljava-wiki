In order to install PL/Java for use with a standard JVM on a Posix (Linux or other *nix) system, you must first make sure that a Java Runtime Environment (JRE) of version 1.4.2 or later is installed on your machine. A Java Development Kit (JDK) is ok too of course, since it includes a JRE.   As of April 2010, JDK 1.6 is not yet supported for compiling, so ideally, use a late 1.5.0_ release like 1.5.0_21.    Client applications can use any Java, of course.

References to JRE_HOME or &lt;jre_home&gt; below means the installation directory of your JRE. In case you have a JDK the JRE_HOME should be equal to $JDK_HOME/jre.

> ##Using GCJ##
> All pre-compiled binaries must run using a standard JVM but the PL/Java can also be compiled and linked using GNU GCJ.
>
> Please note that all use of GCJ should be regarded as experimental. Due to limitations in the implementation of java.security in some versions of GCJ the PL/Java Trusted Language implementation is, in fact, not trusted. At present this applies regardless of GCJ version since the code is conditionally compiled.
>
> None of the settings described here are needed when GCJ is used since the GCJ shared objects are found using the standard loader configuration and since the pljava.jar is compiled and linked with the binary.
>
> See Compile from source for more info on how to compile a GCJ version.

* Download PL/Java from the Download Page and unzip it in a directory of your choice.
* Edit the postgresql.conf file and add (or modify) the following settings (this example assume you used /usr/share/pljava as the installation directory)

```properties
dynamic_library_path = '$libdir:/usr/share/pljava'
custom_variable_classes = 'pljava'
pljava.classpath = '/usr/share/pljava.jar'
```
Please note that the dynamic_library_path setting is not needed if the pljava.dll is installed in the &lt;pgsql-home&gt;/lib directory since it will be searched by default.
* Ensure that the JRE shared libraries can be found by the Postmaster backend. This can be done in different ways depending on platform. Here are two suggestions (the i386 directory will of course be named something suitable for your CPU architecture):
    1. Setting the LD_LIBRARY_PATH environment will work on most Linux/Unix platforms. A good place to do this is usually the file /etc/sysconfig/pgsql/postgresql:
    ```sh
    export LD_LIBRARY_PATH=$JRE_HOME/lib/i386:$JRE_HOME/lib/i386/client
    ```
    Apparently, on some Linux platforms you will also need to append
    ```sh
    :$JRE_HOME/lib/i386/native_threads
    ```
    1. Add a file named postgresql to the directory /etc/ld.so.conf.d that looks like this:
    ```sh
    <jre-home>/lib/i386
    <jre-home>/lib/i386/clients
    ```
    The &lt;jre-home&gt; must be expanded. Then use /sbin/ldconfig to make this new setting effective.
* Restart the Postmaster so that all settings take effect.
* Run the PL/Java Deployer program to install the language handler and the PL/Java specific functions and tables in the database.

##zlib conflict##
On some platforms using Java versions, notably 1.4.2 with a patch level below 6 or 1.5.0 with a patch level below 5, there will be a conflict between the libzip.so included in the JRE and the libz.so used by PostgreSQL (the JRE libzip.so includes a libz.so). The symptom is an InternalError in the java.util.zip.Inflater.init when an attempt is made to load the first class. If you encounter this error you will need to upgrade to a newer Java. You don't need to upgrade to Java 1.5.0. A Java 1.4.2 with patch levels higher than 5 will work.
##64 bit Notes##
If you are running a 64 bit PostgreSQL server, you must use a 64 bit JDK to build and install PL/Java.  On many Unix platforms, Sun JDK requires you install the 32 bit jDK and the 64 bit JDK together. Of course, PostgreSQL clients can use any JRE or JDK.
##Solaris Notes##
On Solaris Sparc platforms, the 64 bit JVM is in $JAVA_HOME/bin/sparcv9 and the libjvm.so needed for the JNI interface is in $JAVA_HOME/jre/lib/sparcv9/server.   The Solaris prebuilt PostgreSQL packages were compiled with Sun Studio 11, you can use 11 or 12.  This compiler is available for free from Sun, and installs to /opt/SUNWSpro/bin by default.   I had to edit the CC = in Makefile.global as my compiler path differed from the original PostgreSQL packagers path.

As of PostgreSQL 8.4.3 the 'official' releases for Solaris *64 bit* have the wrong include/pg_config.h and lib/64/pgxs/src/Makefile.global.  Attempts at hacking this will fail.   You will need to build your own PostgreSQL from source, or somehow acquire these two missing files from the original build.
