##Installing PL/Java as an Extension##

It is fairly straightforward to deploy PL/Java as an extension.

1. Build PL/Java as discussed in [[Building PL/Java]]

2. Copy the shared library to the PostgreSQL lib directory

```{sh}
cp ./pljava-so/target/nar/pljava-so-0.0.2-SNAPSHOT-i386-Linux-gpp-shared/lib/i386-Linux-gpp/shared/libpljava-so-0.0.2-SNAPSHOT.so /usr/lib/postgresql/9.4/lib/pljava.so
```

3. Copy the basic jar file to the PostgreSQL extension directory.

```{sh}
cp ./pljava/target/pljava-0.0.2-SNAPSHOT.jar /usr/share/postgresql/9.4/extension/pjlava-1.4.4.jar
```

Note that shared libraries drop their version number but shared resources do not. I have also changed the version number to reflect the version number of PL/Java (1.4.4) instead of the one given in the maven pom.xml file.

4. Copy and rename the installation script to the extension directory. Note that the filename contains two dashes, not one.

```{sh}
cp ./src/sql/install.sql /usr/share/postgresql/9.4/extension/pljava--1.4.4.sql
```

5. Add one line to the top of that file.

```{sh}
SET PLJAVA.CLASSPATH='/usr/share/postgresql/9.4/extension/pljava-1.4.4.jar';

CREATE SCHEMA sqlj;
GRANT USAGE ON SCHEMA sqlj TO public;
```

6. Create the extension control file (/usr/share/postgresql/9.4/extension/pljava.control)

```{sh}
# pljava extension
comment = 'PL/Java bundled as an extension'
default_version = '1.4.4'
relocatable = false
```

7. Finally make sure that the java shared library jvm.so is visible. On my system I created the file `/etc/ld.so.conf.d/i386/linux-java.conf` with a single line:
    
```{sh}
/usr/lib/jvm/java-8-openjdk-i386/jre/lib/i386/server
```

and now run

```{sh}
$ sudo ldconfig
```

You can now load PL/java as an extension.

```{sh}
bgiles=# create extension pljava;
CREATE EXTENSION
bgiles=#
```
