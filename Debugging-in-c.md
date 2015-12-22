# Debugging PL/Java C code

## Ensure the native code is compiled for debugging

Debugging is much more pleasant when the C code has been compiled with
debugging information included. Edit the `pljava-so/pom.xml` file, find
the `<c>...</c>` section, and add `<debug>true</debug>:

```xml
 <configuration>
  ...
  <c>
   ...
   <debug>true</debug>
   <defines>
    ...
   </defines>
   ...
  </c>
  ...
 </configuration>
```
Save the `pom.xml` file and rebuild PL/Java (or just the `pljava-so`
subproject, to save time).

## Start PL/Java and attach a debugger

Start psql and set the PL/Java debug flag, and issue a call to some Java
function.

```sql
 set pljava.debug to on;
 select sqlj.get_classpath();
```
You will see a message resembling this:

```
INFO:  Backend pid = 2830. Attach the debugger and set pljavaDebug
to false to continue
```

Use another window and attatch gdb or another debugger.  

```sh
 gdb <full path to the postgres executable> <your Backend pid>
```
The debugger will break into the PL/Java code while it is in a dummy loop. You
can break this loop by setting the global variable `pljavaDebug` to false. You
then have the ability to set breakpoints etc. before you continue execution.  

```gdb
 (gdb) set pljavaDebug=0
 (gdb) <set breakpoints etc. here>
 (gdb) cont
```
That's it!
