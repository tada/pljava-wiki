# Logging in PL/Java

PL/Java uses the standard java.util.logging.Logger Hence, you can write things
like:

```java
Logger.getAnonymousLogger().info(
    "Time is " + new Date(System.currentTimeMillis()));
```
At present, the logger is hardwired to a handler that maps the state of
the PostgreSQL configuration setting `log_min_messages` to a valid Logger level
and that outputs all messages using the backend function `ereport()`.

Importantly, Java's Logger methods can quickly discard any message logged at a
finer level than the one that was mapped from PostgreSQL's setting _at the time
PL/Java was first used in the current session_. Such messages never even get
as far as `ereport()`, even if the PostgreSQL setting is changed later.

So, if expected messages from Java code are not showing up, be sure that the
setting in PostgreSQL, at the time of PL/Java's first use in the session, is
fine enough that Java will not throw the messages away. Once PL/Java has
started, the settings can be changed as desired and will control, in the
usual way, what `ereport` does with the messages PL/Java delivers to it.

Through PL/Java 1.5.0, only the `log_min_messages` setting is used to set
that Java cutoff level. Starting with 1.5.1, the cutoff level in Java is set
(still only once at PL/Java startup) based on the finer of `log_min_messages`
and `client_min_messages`.

The following mapping applies between the Logger levels and the PostgreSQL
backend levels:

<table>
<tr><th>java.util.logging.Level</th><th>PostgreSQL level</th></tr>
<tr><td>SEVERE</td><td>ERROR</td></tr>
<tr><td>WARNING</td><td>WARNING</td></tr>
<tr><td>INFO</td><td>INFO</td></tr>
<tr><td>FINE</td><td>DEBUG1</td></tr>
<tr><td>FINER</td><td>DEBUG2</td></tr>
<tr><td>FINEST</td><td>DEBUG3</td></tr>
</table>

See [[Thoughts on logging]] for likely future directions in this area.
