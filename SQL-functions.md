# Functions in the sqlj schema

<a id="install_jar"></a>
## install_jar

The `install_jar` command loads a jarfile from a location appointed by an URL
into the SQLJ jar repository. It is an error if a jar with the given name
already exists in the repository.
#### Usage

`SELECT sqlj.install_jar(<jar_url>, <jar_name>, <deploy>);`

<table>
<tr><th>Parameter</th><th>Description</th></tr>
<tr><td>jar_url</td><td>
The URL that denotes the location of the jar that should be loaded
<tr><td>jar_name</td><td>
This is the name by which this jar can be referenced once it has been loaded
<tr><td>deploy</td><td>
True if the jar should be deployed according to a deployment descriptor, false
otherwise</td></tr>
</table>

<a id="replace_jar"></a>
## replace_jar

The `replace_jar` command will replace a loaded jar with another jar.
Use it to update already loaded files. It's an error if the jar is not found.

#### Usage

`SELECT sqlj.replace_jar(<jar_url>, <jar_name>, <redeploy>);`

<table>
<tr><th>Parameter</th><th>Description</th></tr>
<tr><td>jar_url</td><td>
The URL that denotes the location of the jar that should be loaded.
<tr><td>jar_name</td><td>The name of the jar to be replaced.
<tr><td>redeploy</td><td>
True if the jar should be undeployed according to the deployment descriptor of
the old jar and deployed according to the deployment descriptor of the new jar,
false otherwise. </td></tr>
</table>

<a id="remove_jar"></a>
## remove_jar

The `remove_jar` command will drop the jar from the jar repository.
Any classpath that references this jar will be updated accordingly. It's an
error if the jar is not found.

#### Usage

`SELECT sqlj.remove_jar(<jar_name>, <undeploy>);`

<table>
<tr><th>Parameter</th><th>Description</th></tr>
<tr><td>jar_name</td><td>The name of the jar to be removed.
<tr><td>undeploy</td><td>
True if the jar should be undeployed according to a deployment descriptor,
false otherwise. </td></tr>
</table>

<a id="get_classpath"></a>
## get_classpath

The `get_classpath` command will return the classpath that has been defined for
the given schema. NULL is returned if the schema has no classpath. It's an
error if the given schema does not exist.

#### Usage

`SELECT sqlj.get_classpath(<schema>);`

<table>
<tr><th>Parameter</th><th>Description</th></tr>
<tr><td>schema</td><td>The name of the schema</td></tr>
</table>

<a id="set_classpath"></a>
## set_classpath

The `set_classpath` command will define a classpath for the given schema. A
classpath consists of a colon separated list of jar names. It's an error if the
given schema does not exist or if one or more jar names references nonexistent
jars.

#### Usage

`SELECT sqlj.set_classpath(<schema>, <classpath>);`

<table>
<tr><th>Parameter</th><th>Description</th></tr>
<tr><td>schema</td><td>The name of the schema.
<tr><td>classpath</td><td>The colon separated list of jar names.</td></tr>
</table>

<a id="add_type_mapping"></a>
## add_type_mapping

The `add_type_mapping` command installs a mapping between a SQL type and a Java
class. Once the mapping is in place, parameters and return values will be
mapped accordingly. Please read [[Mapping an SQL type to a Java class]] for
detailed information.

#### Usage

`SELECT sqlj.add_type_mapping(<sql type>, <java class>);`

<table>
<tr><th>Parameter</th><th>Description</th></tr>
<tr><td>sql type</td><td>The name of the SQL type. The name can be qualified with a schema (namespace). If the schema is omitted, it will be resolved according to the current setting of the search_path.
<tr><td>java class</td><td>The name of the class. The class must be found in the classpath in effect for the current schema </td></tr>
</table>

<a id="drop_type_mapping"></a>
## drop_type_mapping

The `drop_type_mapping` command removes a mapping between a SQL type and a Java
class.

#### Usage

`SELECT sqlj.drop_type_mapping(<sql type>);`

<table>
<tr><th>Parameter</th><th>Description</th></tr>
<tr><td>sql type</td><td>The name of the SQL type. The name can be qualified with a schema (namespace). If the schema is omitted, it will be resolved according to the current setting of the search_path.</td></tr>
</table>
