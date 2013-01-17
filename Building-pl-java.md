## Linux Only ==

### Environment variables ###

Define USE_GCJ to use the GCC Java compiler.  With this option PL/Java does not use a JVM but instead compiles all Java code to native binary.

## Linux/Unix ##

### Environment variables ###

With [Feature Request 1010955](http://pgfoundry.org/tracker/index.php?func=detail&aid=1010955&group_id=1000038&atid=337) it is possible to embed the path to the JVM. This negates the need to have it in the LD_LIBRARY_PATH of the server's process and can simplify installation.

Which of the following linker options work will depend on your compiler and platform.

Define USE_LD_RPATH as _n_ where _n_ is

1. Adds -rpath _path to libjvm.so_ to the linker.
1. Adds -R_path to libjvm.so_ to the linker.
1. Adds -R _path to libjvm.so_ to the linker.
1. Adds -Wl,-R_path to libjvm.so_ to the linker.
1. Adds -Wl,-R,_path to libjvm.so_ to the linker.

An example:
 USE_LD_RPATH=1 make

* Linux: Options 1, 4, 5 work with GCC.
* Solaris: Options 2, 3 work with Solaris Studio and 2-5 with GCC.

== Windows ==

_Empty._

== OS/2 ==

_Empty._