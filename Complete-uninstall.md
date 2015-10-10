In order to completely uninstall PL/Java you need to have super user privileges on the database. Here's how you do it.<ol>
<li>See to that no transaction is active that make use of PL/Java functions.</li>
<li>Edit the <data dir>/postgresql.conf file and remove the entries that are associated with PL/Java. Typically the dynamic_library_path, custom_variable_classes, and any variable that starts with 'pljava.'.</li>
<li>Issue a pg_ctl reload -D <data dir> on the database to make the backend aware of the changes.</li>
<li>Use the Deployer with the -remove option. This will remove the sqlj schema from the database and all jars that you have loaded using the -install or -reinstall option.<br/>
(An alternative approach is to run the <PL/Java installation directory>/uninstall.sql script)</li>
<li>Remove the <PL/Java installation directory> and all that it contains.</li>
</ol>