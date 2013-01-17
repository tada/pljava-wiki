The PL/Java Source distribution contains a couple of rudimentary tests. The tests are divided into two jar files. One is the client part found in the test.jar. It contains some methods that executes SQL statements and prints the output (all contained there can of course also be executed from psql or any other client). The other is the examples.jar which contains the sample code that runs in the backend. The latter must be installed in the database in order to function. An easy way to do this is to use psql and issue the command:
```sql
SELECT sqlj.install_jar('file:///some/directory/examples.jar', 'samples',  true);
```
Please note that the deployment descriptor stored in examples.jar will attempt to create the schema javatest so the user that executes the sqlj.install_jar command must have permission to do that.

Once loaded, you must also set the classpath used by the PL/Java runtime. This classpath is set per schema (namespace). A schema that lacks a classpath will default to the classpath that has been set for the public schema. The tests will use the schema javatest. To define the classpath for this schema, simply use psql and issue the command:
```sql
SELECT sqlj.set_classpath('javatest', 'samples');
```
The first argument is the name of the schema, the second is a colon separated list of jar names. The names must reflect jars that are installed in the system.

NOTE: If you don't use schemas, you must still issue the set_classpath command to assign a correct classpath to the 'public' schema. This can only be done by a super user.

Now, you should be able to run the client test application:

```sh
java -cp <path including the jdbc driver and test.jar> org.postgresql.pljava.test.Tester
```