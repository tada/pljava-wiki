# Exception handling

You can catch and handle an exception in the PostgreSQL back-end just like any
other exception. The back-end `ErrorData` structure is exposed as a property in
a `ServerException` class derived from `java.sql.SQLException`, and the Java
try/catch mechanism is synchronized with the back-end mechanism.

*Note: for several reasons (see [[Thoughts on logging]] for background),
referring to `ServerException` and `ErrorData` from your code is not
currently recommended, and in the future may become impossible. An improved
mechanism is expected in a future release. Until then, using only the
standard Java API of `java.sql.SQLException` and its standard attributes
(such as `SQLState`) is recommended wherever possible.*

PL/Java will always catch exceptions that you don't. They will cause a
PostgreSQL error and the message is logged using the PostgreSQL logging
utilities. The stack trace of the exception will also be printed if the
PostgreSQL configuration parameter `log_min_messages` is set to `DEBUG1`
or lower.

### Important Note:

You will not be able to continue executing back-end functions until your
function has returned and the error has been propagated when the back-end has
generated an exception unless you have used a save-point. When a save-point is
rolled back, the exceptional condition is reset and execution can continue.
