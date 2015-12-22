# Savepoints

PostgreSQL savepoints are exposed using the standard `setSavepoint()` and
`releaseSavepoint()` methods on the `java.sql.Connection` interface. Two
restrictions apply:

* A savepoint must be rolled back or released in the function where it was set.
* A savepoint must not outlive the function where it was set.

"Function" here refers to the PL/Java function that is called from SQL.
The restrictions do not prevent the Java code from being organized into
several methods, but the savepoint cannot survive the eventual return
from Java to the SQL caller.
