## Linux Only ##

### Maven Builds ###

It is easy to build PL/Java on Linux systems.

1. Clone the PL/Java project from github (https://github.com/tada/pljava).

2. Install java and maven.

_Ubuntu / Debian_

```{sh}
$ sudo install openjdk-8-jdk
$ sudo install maven
```

_RedHat / Fedora / CentOS_

```{sh}
$ sudo yum install java-1.8.0-openjdk
$ sudo yum install apache-maven
```

In the root directory of the cloned project run ```mvn install```. This will automatically download a large number of files the first time you run it but that is normal. You will also see a number of [ERROR] messages but that is acceptable as long as the build succeeds.

```{sh}
[INFO] Scanning for projects...
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Build Order:
[INFO] 
[INFO] PostgreSQL pl/java
[INFO] PL/JAVA API
[INFO] pl/java implementation
[INFO] pl/java deploy
[INFO] pl/java Ant Tasks
[INFO] pl/java examples
[INFO] pl/java server side library
[INFO] pl/java packaging
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building PostgreSQL pl/java 0.0.2-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-install-plugin:2.3:install (default-install) @ pljava.app ---
[INFO] Installing /home/bgiles/Documents/src/pljava/pom.xml to /home/bgiles/.m2/repository/org/postgresql/pljava.app/0.0.2-SNAPSHOT/pljava.app-0.0.2-SNAPSHOT.pom
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building PL/JAVA API 0.0.2-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 

(100 lines skipped...)

[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building pl/java packaging 0.0.2-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-antrun-plugin:1.7:run (pljava package distribution) @ packaging ---
[INFO] Executing tasks

main:

init:
[loadresource] Do not set property naraol as its length is 0.

package-targz:
      [tar] Building tar: /home/bgiles/Documents/src/pljava/packaging/target/docs.tar
      [tar] Building tar: /home/bgiles/Documents/src/pljava/packaging/target/pljava.tar
     [gzip] Building: /home/bgiles/Documents/src/pljava/packaging/target/pljava-pg9.4-${naraol}.tar.gz
[INFO] Executed tasks
[INFO] 
[INFO] --- maven-install-plugin:2.3:install (default-install) @ packaging ---
[INFO] Installing /home/bgiles/Documents/src/pljava/packaging/pom.xml to /home/bgiles/.m2/repository/org/postgresql/packaging/0.0.2-SNAPSHOT/packaging-0.0.2-SNAPSHOT.pom
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO] 
[INFO] PostgreSQL pl/java ................................ SUCCESS [1.724s]
[INFO] PL/JAVA API ....................................... SUCCESS [22.776s]
[INFO] pl/java implementation ............................ SUCCESS [20.429s]
[INFO] pl/java deploy .................................... SUCCESS [2.780s]
[INFO] pl/java Ant Tasks ................................. SUCCESS [3.847s]
[INFO] pl/java examples .................................. SUCCESS [7.096s]
[INFO] pl/java server side library ....................... SUCCESS [1:13.899s]
[INFO] pl/java packaging ................................. SUCCESS [5.328s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 2:26.226s
[INFO] Finished at: Sat Aug 08 11:45:04 MDT 2015
[INFO] Final Memory: 23M/230M
[INFO] ------------------------------------------------------------------------
```

This produces two artifices of interest. The linux shared library is at

```{sh}
./pljava-so/target/nar/pljava-so-0.0.2-SNAPSHOT-i386-Linux-gpp-shared/lib/i386-Linux-gpp/shared/libpljava-so-0.0.2-SNAPSHOT.so
```

The core .jar file is at

```{sh}
./pljava/target/pljava-0.0.2-SNAPSHOT.jar
```


### Environment variables ###

(outdated?)

Define USE_GCJ to use the GCC Java compiler.  With this option PL/Java does not use a JVM but instead compiles all Java code to native binary.

## Linux/Unix ##

### Environment variables ###

With [Feature Request 1010955](http://pgfoundry.org/tracker/index.php?func=detail&aid=1010955&group_id=1000038&atid=337) it is possible to embed the path to the JVM. This negates the need to have it in the LD_LIBRARY_PATH of the server's process and can simplify installation.

Which of the following linker options work will depend on your compiler and platform.

Define USE_LD_RPATH as _n_ where _n_ is

1. Adds -rpath <i>&lt;path to libjvm.so&gt;</i> to the linker.
1. Adds -R <i>&lt;path to libjvm.so&gt;</i> to the linker.
1. Adds -R<i>&lt;path to libjvm.so&gt;</i> to the linker.
1. Adds -Wl,-R <i>&lt;path to libjvm.so&gt;</i> to the linker.
1. Adds -Wl,-R,<i>&lt;path to libjvm.so&gt;</i> to the linker.

An example:
 USE_LD_RPATH=1 make

* Linux: Options 1, 4, 5 work with GCC.
* Solaris: Options 2, 3 work with Solaris Studio and 2-5 with GCC.

## Windows ##

_No content provided yet._