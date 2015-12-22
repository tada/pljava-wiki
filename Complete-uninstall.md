In order to completely uninstall PL/Java you need to have super user privileges on the database. Here's how you do it.

0. Get rid of the `sqlj` schema and all objects depending on it.

    * If you installed PL/Java with `CREATE EXTENSION pljava` then drop it
        with `DROP EXTENSION pljava CASCADE`
    * If you installed PL/Java with a `LOAD` command, then drop it with
        `DROP SCHEMA sqlj CASCADE`

    **Caution:** Either command will drop the PL/Java schema and language
    declarations, all jars you may have loaded, all functions and types
    they provided, *and everything else in your database that depends on
    any of those things*.

    You can try either command *without* `CASCADE` first, to see a list
    of what would be dropped.

0. Remove any settings of PL/Java variables (configuration variables with
    names starting with `pljava.`) that you may have changed from their
    defaults.

    * If you had set a variable *var* for a particular database using
        `ALTER DATABASE dbname SET var ...` then reset it using
	`ALTER DATABASE dbname RESET var`.

    * If you had set it for the whole cluster using
        `ALTER SYSTEM SET var ...` then reset it using
	`ALTER SYSTEM RESET var` and, when you have reset all, use
	`SELECT pg_reload_conf()`.

    * If you had set PL/Java variables by editing the configuration file
        (particularly on PostgreSQL before 9.2, where this is the only
	available method), remove the settings from the file, then use
	`SELECT pg_reload_conf()` in SQL.

    * The variable `dynamic_library_path` is not specific to PL/Java, but
        if you added a directory to it for the sake of PL/Java, undo that.

0. Remove the PL/Java files from the file system.

    * If you installed PL/Java with a package manager, uninstall it
        the same way.
    * Otherwise, remove the installed files from wherever you installed them.

