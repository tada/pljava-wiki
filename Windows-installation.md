#Prerequisites#
In order to install PL/Java on Windows, you must first make sure that a Java Runtime Environment (JRE) of version 1.4.2 or later is installed on your machine. A Java Development Kit (JDK) is ok too of course, since it includes a JRE.
#Using the Windows Installer#
PL/Java is bundled with the PostgreSQL Windows Installer. It will install PL/Java under your PostgreSQL installation and set up all needed configurations and run the PL/Java deployer.
#Installing manually.#
* Download PL/Java from the Download Page and unzip it in a directory of your choice.
* Edit the postgresql.conf file and add (or modify) the following settings (this example assume you used C:\\utils\\pljava as the installation directory). Note the double backslashes:<br/>
```properties
dynamic_library_path ='$libdir;C:\\\\utils\\\\pljava'
custom_variable_classes = 'pljava'
pljava.classpath='C:\\\\utils\\\\pljava\\\\pljava.jar'
```
Please note that the dynamic_library_path setting is not needed if the pljava.dll is installed in the &lt;pgsql-home&gt;\\lib directory since it will be searched by default.
* Modify the PATH setting used by the PostgreSQL backend (if it runs as a service, you will normally change the System Environment setting) so that it contains the following two entries:
```bat
%JRE_HOME%\\bin;%JRE_HOME%\\bin\\client
```
The JRE_HOME in this case appoints the installation directory of your JRE. If you have a JDK installed, then the JRE_HOME will be equal to %JAVA_HOME%\\jre.
* Restart the Postmaster so that all settings take effect.
* Run the PL/Java Deployer program to install the language handler and the PL/Java specific functions and tables in the database.
