# Thoughts on logging

_Note: this page does not describe how PL/Java currently works, except in
the "Background" part. It is a proposal for further development._

_Also, to anyone reading this for review or comment, a lot of this material
may be more familiar than it is to me. If I seem to describe it in excessive
detail, please regard that as my effort to have it straight in my own head._

_Also also, to some it may seem strange that I use phrases like "log event"
without insisting on any essential difference between_ thrown exceptions
_and_ calls on loggers. _It's true, I'm not marking any such essential
difference, and I hope, before this is done, that won't seem so strange._

So, here goes.

_... logging isn't particularly magical._  
—Dave Cramer

_... it's what you do with it._  
—variously attributed

## Background

### PostgreSQL has supremely good log messages.

It even has a [style guide][stygd] for writing them, and it pays off
in the well-known quality and helpfulness of PostgreSQL messages.

[stygd]: http://www.postgresql.org/docs/current/static/error-style-guide.html

Part of the excellence of PostgreSQL's messages can be found in their rich
structure. A message is not a blob of text with whatever details seemed
useful while writing the code. It is a structured record with information
serving several specific purposes and at several distinct levels of detail:

item            |pq|PL/pgSQL            |pgjdbc (+ -ng Notice)|pgjdbc-ng Exc |PL/Java
----------------|---|--------------------|---------------------|--------------|-----------------
elevel          |S |                    |getSeverity          |              |getErrorLevel
sqlstate        |C |RETURNED_SQLSTATE   |getSQLState  getCode |getSQLState   |getSqlState
message         |M |MESSAGE_TEXT        |getMessage           |getMessage    |getMessage
detail          |D |PG_EXCEPTION_DETAIL |getDetail            |              |getDetail
hint            |H |PG_EXCEPTION_HINT   |getHint              |              |getHint
context         |W |PG_EXCEPTION_CONTEXT|getWhere             |              |getContextMessage
schema_name     |s |SCHEMA_NAME         |getSchema            |getSchema     |
table_name      |t |TABLE_NAME          |getTable             |getTable      |
column_name     |c |COLUMN_NAME         |getColumn            |getColumn     |
datatype_name   |d |PG_DATATYPE_NAME    |getDatatype          |getDatatype   |
constraint_name |n |CONSTRAINT_NAME     |getConstraint        |getConstraint |
cursorpos       |P |                    |getPosition          |              |getCursorPos
internalpos     |p |                    |getInternalPosition  |              |getInternalPos
internalquery   |q |                    |getInternalQuery     |              |getInternalQuery
filename        |F |                    |getFile              |              |getFilename
lineno          |L |                    |getLine              |              |getLineno
funcname        |R |                    |getRoutine           |              |getFuncname
output_to_server|  |                    |                     |              |isOutputToServer
output_to_client|  |                    |                     |              |isOutputToClient
show_funcname   |  |                    |                     |              |isShowFuncname
saved_errno     |  |                    |                     |              |getSavedErrno
hide_stmt       |  |                    |                     |              |
hide_ctx        |  |                    |                     |              |
domain          |  |                    |                     |              |
context_domain  |  |                    |                     |              |


The `libpq` on-the-wire protocol preserves this structure, sending these
components (the ones with `pq` codes) distinctly and intact to the front end.
This gives client code enormous flexibility to catch and handle conditions
appropriately. If the condition has to be logged or reported to a user, it can
be shown at any appropriate level of detail, or even with a user interface that
permits drilling down from generalities to specifics.

How awesome is that? Consider this: I have seen, with my own eyes,
non-technical users entering stuff into a PostgreSQL database (using
something as generic as LibreOffice Base as the front end) have an
error dialog pop up, _read it_, understand what had to be corrected
in the entry, and recover on their own.

I challenge anyone who has had support experience to tell me _that_ ain't magic.

By and large, messages in PostgreSQL really are _that_ good.

### Original log event life cycle

A message that originates in the PostgreSQL backend proper begins as a call
to `ereport` or `elog` in [elog.c][elogc]. The rules there are just a bit
_special_:

0. If the message has any severity _below_ `ERROR` (so, `DEBUG5`, `DEBUG4`,
    `DEBUG3`, `DEBUG2`, `DEBUG1`, `LOG`, `COMMERROR`, `INFO`, `NOTICE`, or
    `WARNING`) __or__ _above_ `ERROR` (so, `FATAL`, `PANIC`), it gets written
    immediately to logs / reported to the front end (according to
    the `log_min_messages` and `client_min_messages` settings), and then
    control returns to the call site if it was below `ERROR`, and does not
    return if it was above.

0. If the severity is _exactly_ `ERROR`, it gets _thrown_ PostgreSQL-style, and
    can be caught in `PG_TRY`/`PG_CATCH` constructs. The [elog][elogc] code
    does _not_ send it to the server log or the front end at all at that point,
    but only when (if ever) it bubbles up to `PostgresMain` without having been
    handled. If it gets caught and handled, then any logging becomes the
    responsibility of whatever code caught it.

[elogc]: http://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/utils/error/elog.c

### Handling logged events in front-end code

Events from the server that arrive at the front end show up in a form the
front-end code can inspect and choose how to handle, either by polling for
them (libpq `PQresultErrorField`, JDBC `getWarnings` if less severe than
`ERROR`), or catching an exception (JDBC if severity is `ERROR`).

This code in turn might want to log an event (whether one received from the
backend as just described, or one originating in the front-end code itself).
To do that, it will probably use some convenient library available to it,
such as `java.util.logging` in Java.

For code that is really running in a front end, that's the end of the story,
but for code running in a backend PL, the story has only begun.

### Handling logged events in a back-end PL

The first requirements for server-side code are the same as for the front end:
it should be able to intercept, examine, and handle or not handle as
appropriate, events that originate during its calls into the backend.
PL/pgSQL makes a good example, with the [trapping of errors][plpgscatch]
built into the language, and [inspection of (most) elements][plpgsdiag] of
the structure. Naturally, whatever isn't caught in PL/pgSQL code should
continue propagating outward with all its structured information intact.

In PL/pgSQL, all of that applies to events with the exact severity `ERROR`:
warnings/notices/etc. are invisible to PL/pgSQL code, as the
[elog][elogc] logic sends those right out from under the PL and straight
to the front-end client (conditioned only on the `client_min_messages`
setting). That might not be always ideal: there can be a tension between ease
of development/troubleshooting, favoring lots of logging, and confidentiality
demands, which could require that some messages be edited or suppressed, which
the PL code can't do if they zip right past it.

[plpgscatch]: http://www.postgresql.org/docs/current/static/plpgsql-control-structures.html#PLPGSQL-ERROR-TRAPPING
[plpgsdiag]: http://www.postgresql.org/docs/current/static/plpgsql-control-structures.html#PLPGSQL-EXCEPTION-DIAGNOSTICS

#### As for PL/Java

PL/Java is at parity with PL/pgSQL as far as the ability to catch error events
from the backend (as Java `SQLException`s), or to propagate them up the stack
without information loss if they are not caught, or are caught and rethrown
without change.

Internally, it has good provisions for examining individual properties of
the event (currently not including the schema, table, column, datatype, and
constraint names that [appeared in 9.3][fnames]). However, these provisions
aren't quite exposed yet to ordinary developers writing PL/Java code. Although
[[documented|Exception handling]] in the wiki, they aren't accessible to
PL/Java code compiled normally against `pljava-api.jar`; it would have to
be compiled against the full `pljava.jar` and refer explicitly to
[`ServerException`][servx] and [`ErrorData`][edata] in the
`org.postgresql.pljava.internal` package.

[fnames]: http://git.postgresql.org/gitweb/?p=postgresql.git;a=blobdiff;f=src/include/utils/elog.h;h=d5fec89a4801481d0494c34cd38e72653d3da99a;hp=5e937fb10c3211b2c12d0ec2d7ee71fe506cb7f8;hb=991f3e5ab3f8196d18d5b313c81a5f744f3baaea;hpb=89d00cbe01447fd36edbc3bed659f869b18172d1
[servx]: http://tada.github.io/pljava/pljava/apidocs/index.html?org/postgresql/pljava/internal/ServerException.html
[edata]: http://tada.github.io/pljava/pljava/apidocs/index.html?org/postgresql/pljava/internal/ErrorData.html

(In passing, the current design where the magic only happens for a single
`ServerException` subclass of `SQLException` stands in the way of implementing
the [categorized exceptions][catex] for JDBC 4.0.)

[catex]: http://www.javaspecialists.eu/archive/Issue138.html

This is one area where a good API should be worked out and published.
PL/Java provides a JDBC interface to the backend, because that is how
the SQL/JRT standard is written, which PL/Java is meant to implement.
As far as JDBC is concerned, access to these implementation-specific error
details would be an extension, such as might be accessed
by [`unwrap`][unwrap] on a standard JDBC object. Being committed to a JDBC
interface
to PostgreSQL, it would be ideal to agree on the details with the other,
front-end JDBC interfaces to PostgreSQL, as front-ends also receive finely
structured PostgreSQL error details that client code may want to examine.

(Again in passing, the fact that PL/Java presents a JDBC interface could raise
hopes that PL/Java functions are also able to examine warnings using the
standard [JDBC mechanism][jdbcw]. At the moment, they aren't, and the
`PG_TRY`/`PG_CATCH` constructs aren't enough to fix that, because only severity
level `ERROR` is handled that way. PL/Java would have to also use the
`emit_log_hook` in order to present a behavior analogous to JDBC on the
front end. It would then pull ahead of PL/pgSQL on that dimension.)

[unwrap]:  http://docs.oracle.com/javase/7/docs/api/index.html?java/sql/Wrapper.html#unwrap(java.lang.Class)
[jdbcw]: http://docs.oracle.com/javase/7/docs/api/index.html?java/sql/ResultSet.html#getWarnings()

#### How do the other PostgreSQL JDBCs give access to error details?

##### pgjdbc

If you have an `SQLException` _and_ it can be cast to
[`org.postgresql.util.PSQLException`][psqle], then you can call
`getServerErrorMessage()` on it, and get a [`ServerErrorMessage`][sem].
Same deal if you have an `SQLWarning` that is castable to
[`org.postgresql.util.PSQLWarning`][psqlw]. (None of this is
exactly trumpeted in the [docs][pgjdbcdocs], and as you can see, the
`PSQLException` and `PSQLWarning` links above are to `privateapi` pages,
though the classes are `public` and accessible.)

Just as in PL/Java, the design here with a single `PSQLException` class is an
obstacle to moving forward with the categorized exceptions in JDBC 4.0.

[psqle]: https://jdbc.postgresql.org/development/privateapi/index.html?org/postgresql/util/PSQLException.html
[psqlw]: https://jdbc.postgresql.org/development/privateapi/index.html?org/postgresql/util/PSQLWarning.html
[sem]: https://jdbc.postgresql.org/documentation/publicapi/index.html?org/postgresql/util/ServerErrorMessage.html
[pgjdbcdocs]: https://jdbc.postgresql.org/documentation/head/index.html

##### pgjdbc-ng

In `pgjdbc-ng`, categorized exceptions are partially implemented: at least
there is one [`PGSQLIntegrityConstraintViolationException`][pgsicve], and one
[`PGSQLSimpleException`][pgsse] for everything else. To allow for multiple
categories, these share a common _interface_, [`PGSQLExceptionInfo`][pgsei].
Calling code does not need to test for a bunch of implementation-specific
class names, but can simply catch JDBC exceptions by their standard `java.sql`
names, and test for castability to a single interface.

Interestingly, the interface gives access _only_ to the column, constraint,
datatype, schema, and table names available from PostgreSQL 9.3 onward (exactly
the five things PL/Java currently _doesn't_ expose!) and none of the much more
anciently supported detail, hint, context, etc. And `pgjdbc-ng` doesn't supply
any `SQLWarning` subclass that implements it.

_All_ of the elements the protocol can forward are available on a different
object, [`com.impossibl.postgres.protocol.Notice`][notice], but I am not sure
user code has any way to get one. The classes that use it seem fairly internal.

Unlike both PL/Java's [`ErrorData`][edata] and `pgjdbc`'s
[`ServerErrorMessage`][sem], in `pgjdbc-ng` both the
[`PGSQLExceptionInfo`][pgsei] and the [`Notice`][notice] are mutable,
providing setter methods as well as getters ... leading naturally into
the next section.

[pgsicve]: http://impossibl.github.io/pgjdbc-ng/apidocs/0.6/index.html?com/impossibl/postgres/jdbc/PGSQLIntegrityConstraintViolationException.html
[pgsse]: http://impossibl.github.io/pgjdbc-ng/apidocs/0.6/index.html?com/impossibl/postgres/jdbc/PGSQLSimpleException.html
[pgsei]: http://impossibl.github.io/pgjdbc-ng/apidocs/0.6/index.html?com/impossibl/postgres/api/jdbc/PGSQLExceptionInfo.html#method_summary
[notice]: http://impossibl.github.io/pgjdbc-ng/apidocs/0.6/index.html?com/impossibl/postgres/protocol/Notice.html#method_summary

### Originating loggable events

Whether running client-side or in a server-side PL, it's often helpful to look
at the details of events from below, as the last section explored. But for a
server-side PL, it doesn't stop there, because server-side code is usually
implementing logic that may have its own events to report, and as far as the
client-side is concerned, _those are just more events from the backend_.
Ideally, a PL function would be a "full citizen" and able to originate events
(or rethrow caught ones wrapped in higher-level descriptions) with the same
structure and quality one expects to see from PostgreSQL itself.

Here again, PL/pgSQL makes a good example. Using [RAISE][], code can generate
an event with direct control of eleven of its most interesting attributes.
(The ones not settable from PL/pgSQL, cursor positions, line numbers, and such,
are at a level of detail few PL/pgSQL functions would want to work at anyway.)

[RAISE]: http://www.postgresql.org/docs/current/static/plpgsql-errors-and-messages.html

PL/pgSQL has hit a kind of sweet spot with syntax that is so easy and clear
it invites writing good messages that an ultimate user could find helpful.
A statement like:

    RAISE invalid_text_representation USING
      MESSAGE = 'Unrecognized prefix in telephone number ' || tno,
      DETAIL = 'The digits at the start of the number do not match any known '
               'international number prefix. Are you sure it is right?',
      HINT = 'If you are sure it''s right, ask the IT people if '
             'there is a more recent "ITU-T bulletin 994" they can load. '
             'Meanwhile, you can enter the number with a ! in front, '
             'but it may be flagged on data quality reports until fixed.';

is about as clear as can be in the code, as well as giving anyone who receives
it a fighting chance at understanding what has happened.

#### As for PL/Java

PL/Java is not yet at parity with PL/pgSQL on this dimension. If PL/Java code
_catches_ any exception that began as a PostgreSQL [`ereport`][elogc], that
Java exception will wrap an [`ErrorData`][edata] object; if _that same
exception_ is rethrown, it is transparently turned back into a PostgreSQL
event and continues on its way without information loss. But there is no way
for PL/Java code to _create_ an exception with those properties. At best, it
can create an ordinary `SQLException`, which will turn into a PostgreSQL log
event using its `SQLState` and with its class name and message used as the
`message`.

For any other kind of exception, only `message` is set (from the exception
class name and message), and `SQLState` of `XX000` for "internal error".
When the origin is a Java exception, the severity will always be `ERROR`.

The other way for PL/Java code to originate a log event is to use the
logging API. PL/Java presents mostly Java standard APIs for code to use,
so in this case the API is [`java.util.logging`][jlog], and PL/Java has
wired it so log events created that way are handed off to the PostgreSQL
logging system. (As a side effect of the way that system works, 'logging'
any event with a severity that maps to PostgreSQL `ERROR` turns out to have
the same effect as _throwing_ it, while at any other severity it simply gets
logged.)

When passed on to PostgreSQL, the details include the timestamp, class name
or logger name, the message, and the stack trace of any associated Java
throwable—but at present, all of that ends up strung together in the
`message` attribute of the PostgreSQL event, using a severity mapped from
the [`java.util.logging.Level`][jlvl]. There are some other low-hanging-fruit
mappings that could be made automatically,
like the Java [`SourceClassName`][scn] and [`SourceMethodName`][smn] to
PostgreSQL `filename` and `funcname`, but for the present they are not,
and no other programmatic control over the created log event is yet available
to PL/Java code.

[jlog]: http://docs.oracle.com/javase/7/docs/api/index.html?java/util/logging/package-summary.html
[jlvl]: http://docs.oracle.com/javase/7/docs/api/index.html?java/util/logging/Level.html
[scn]: http://docs.oracle.com/javase/7/docs/api/index.html?java/util/logging/LogRecord.html#getSourceClassName()
[smn]: http://docs.oracle.com/javase/7/docs/api/index.html?java/util/logging/LogRecord.html#getSourceMethodName()

##### Mapping of severity levels

Because there are only seven predefined [`java.util.logging.Level`][jlvl]s and
some of their names are different from PostgreSQL's, PL/Java maps them as
follows:

 | |FINEST|FINER|FINE| | |INFO| |WARNING|SEVERE| | 
---|---|---|---|---|---|---|---|---|---|---|---|---
DEBUG5|DEBUG4|DEBUG3|DEBUG2|DEBUG1|LOG|COMMERROR|INFO|NOTICE|WARNING|ERROR|FATAL|PANIC

The Java level `CONFIG` isn't explicitly mapped, and anything that isn't
explicit will map to the PostgreSQL level `LOG`. For completeness, I've
shown the PostgreSQL levels `FATAL` and `PANIC`, though a good case could be
made that no PL code should ever be allowed to use them.)

#### How can JDBC front-end code originate events?

While the front-end situation may be simpler (there is no need to make
logging interoperate with server code in both directions, as PL/Java must),
some of the same considerations can be carried over. Even on the client end,
JDBC is not _the client_, it's still part of _the stack_. It may originate
its own log messages or throw its own `SQLException`s for reasons other than
events it forwards from the backend. Layers above it see a clean, consistent
picture of "the database stack" when those events are of similar form, no
matter the level they come from.

These days, there could even be another sophisticated layer or three sitting
on top of JDBC and beneath the application code, and it might want to have the
same facilities available to it.

A day that I would like to see—and I think it can be reached—is the day
when a PostgreSQL error can be raised by the backend, caught by PL/Java,
examined in all its structured detail by Java code using some extension
of the JDBC API, rethrown, piped to the frontend JDBC and thrown again to the
client code, caught there, and examined again in the same structured detail
using _the same extended API_.

This becomes even more appealing, and maybe even more achievable, if PL/Java
and a front-end JDBC work toward sharing more code.

##### when throwing

Being JDBC interfaces, both `pgjdbc` and `pgjdbc-ng` throw the standard JDBC
`SQLException` (or, more precisely, subclasses of it), and create instances
of the standard `SQLWarning`, which are collected and polled for, rather than
thrown.

JDBC categorized exceptions are not yet supported by `pgjdbc`, and are
partly supported by `pgjdbc-ng`.

The `pgjdbc-ng` exceptions that implement [`PGSQLExceptionInfo`][pgsei] can be
instantiated from scratch, and can have the column, constraint,
datatype, schema, and table names set, as well as the JDBC standard
`SQLException` attributes.

The `pgjdbc` [`PSQLException`][psqle] and [`PSQLWarning`][psqlw] throwables
can be instantiated from scratch, and can be constructed from a
[`ServerErrorMessage`][sem] that can also be built from scratch, allowing
control over all of the same log event attributes that would be sent to the
front-end for a backend event. However, the only way at present to construct
that `ServerErrorMessage` is to supply a `String` in the exact format of the
`v3` protocol message that would come from the backend to represent the event.
The constructed exception's `message` then is set to the entire result of
`toString` on the `ServerErrorMessage`.

##### when logging

Logging **in `pgjdbc`**, which is older than `java.util.logging`, is done with
the project-specific `org.postgresql.core.Logger`. This simple class identifies
messages with a connection ID prefix, filters them by severity, timestamps
them, and writes them to the DriverManager's LogWriter. Unlike
`java.util.logging`, it doesn't do message formatting or internationalization,
but a separate class [`org.postgresql.util.GT`][gt] (also excluded from the
public API) does both when used in calls
to the logger. It gets translations from [`ResourceBundle`][rb]s, the same
form `java.util.logging` uses.

When logging a `PSQLException` instance that carries a
`ServerErrorMessage`, the result looks much like a message logged by the
backend, because `ServerErrorMessage.toString` produces that form (and it was
entirely stuffed into the `message` attribute of the exception).

[gt]: https://jdbc.postgresql.org/development/privateapi/index.html?org/postgresql/util/GT.html
[rb]: http://docs.oracle.com/javase/7/docs/api/index.html?java/util/ResourceBundle.html

Logging **in `pgjdbc-ng`** is done using `java.util.logging`, as in PL/Java,
and in keeping with the appearance of `java.util.logging`-related API in
JDBC itself [starting in 4.1][gpl]. In the existing examples in the code where a
`NoticeException` or `SQLException` are passed directly to the logger, most
of the available [`Notice`][notice] or [`PGSQLExceptionInfo`][pgsei] will
not be seen, as far as I can see, as `toString` has not been overridden.
[`ErrorUtils`][erut] will create the `SQLException` subclasses using only
the `message` and sqlstate from the original `Notice`. A
[`NoticeException`][nex] holds a reference to the `Notice` it was constructed
from, and has a method to retrieve it, but the exception's message is set using
only the additional `String` passed to its constructor.

[gpl]: http://docs.oracle.com/javase/7/docs/api/index.html?java/sql/Driver.html#getParentLogger()
[nex]: http://impossibl.github.io/pgjdbc-ng/apidocs/0.6/index.html?com/impossibl/postgres/system/NoticeException.html

## What would a nice API look like?

By this point it should be clear why I've been writing about "log events" as
one concept, when I might be expected to talk of log messages and exceptions
as separate things. In PostgreSQL and in JDBC, both are forms the same
information may take as it travels between A and B. It may be passed along
as a message on a log channel, received, and thrown as an exception; something
thrown as an exception can be caught and stuffed onto a log channel, where
"log channel" might mean the `ereport` conveyor in the server code, the
network protocol to the front end, the SQL warnings chain in JDBC, ....

This shapeshifting is not only possible but downright common in PostgreSQL and
JDBC, and especially in PL/Java, where the same event may be batted about
between those two forms repeatedly (how deep can the call stack get with PL/Java
functions making SQL queries that call other functions also made in PL/Java?).
And all of that is just fine as long as the conversion at each step is
information-preserving and reversible.

### An abstract `LogRecord` class (not derived from `Exception`)

We've seen that all three of (`pgjdbc`, `pgjdbc-ng`, `PL/Java`) include a
class of some sort ([`ServerErrorMessage`][sem], [`Notice`][notice], and
[`ErrorData`][edata], respectively) that is meant to carry all the information
about a PostgreSQL log event in its intact structured form, and can be
carried over a log channel or wrapped in an exception, and recovered at the
end of a journey either way.

So, my proposal starts here: there should be such a class, and it should be
documented and available as PostgreSQL extended JDBC API. For this discussion,
I'll call it `LogRecord`. (There will be time for polishing name choices.
There is an existing Java class `LogRecord` but of course the package is
different.)

### An interface for exceptions that carry `LogRecord`s

To allow moving forward with the [categorized exceptions][catex] in
JDBC 4.0, this has to be an interface rather than a common parent class,
and simply has a setter and getter for attaching a `LogRecord` to the
exception. These would ordinarily not be used directly, but rather through
methods of `LogRecord`.

### Different concrete subclasses of `LogRecord`

PL/Java would supply its special concrete implementation that wraps a
native error-data block; a front-end JDBC would supply one (or two) that
are initialized from the on-the-wire protocol. In all cases, there would be
a concrete implementation that can be filled in from scratch.

### An `ereport`-like API for creating a `LogRecord` from scratch

... starting with a static method on `LogRecord` and with method chaining:

```
import static org.postgresql.something.LogRecord.ereport;
...
logrec = ereport(Level.ERROR).errcode(ERRCODE_DIVISION_BY_ZERO)
          .errmsg("You''ve tried to divide {0} by zero", dividend)
	  // .log()   OR
	  // .throwAs(SQLException.class)
```

### Methods for the conversions into/out of log system or exception

Examples `log()` and `throwAs()` were seen above, with `throwAs` a convenience
built on `asException(...)`. If the `LogRecord` has been freshly constructed,
`asException` creates the correct JDBC 4 categorized exception with a reference
to the log record and vice versa. If that has happened already, it just returns
the already created exception object.

The static `fromException()` method does the reverse: if the exception was
created from a `LogRecord` originally (so, it implements the interface and its
log record reference isn't null), just returns that original `LogRecord`. If
not, creates a new `LogRecord` initialized as informatively as possible with
whatever can be gleaned from the exception.

Those methods are what make possible the repeated batting around that an event
might live through on its way from a deep call stack in PL/Java all the way
out to a handler on the front end, without having serious identity crises.
(It will be trickier inside PL/Java than I need to spend time on here, but
should not be prohibitively so.)

### In concert with existing standard API

I propose to converge on [`java.util.logging`][jlog], and for this extended
`LogRecord` class to be derived from the standard [`LogRecord`][logrec].

(Soon below I will touch on how to "converge on `java.util.logging`" without
serious disruption of `pgjdbc`, which currently uses the homegrown logging
class.)

The specialized class will have several extra methods, and some overridden
ones just to give it a reasonable default behavior when treated as an
ordinary `java.util.logging.LogRecord`. For example its overridden
[`getMessage`][gmsg] method may do some formatting by default and return
more information than just the `message` field, while different methods would
be provided for a caller in the know to examine specific individual fields.

None of those differences stop it from being a valid instance of
`java.util.logging.LogRecord`, and it can be passed into the logging system
by calling [`log`][logr] just like any other record. So can other, non-extended
`LogRecord`s and normal calls on the convenience methods of
[`Logger`][lgr], all at the same time. Code ported from other environments,
knowing nothing of the extensions and using the standard logger API will work
fine, and can be mixed with code using the extended features.

Call sites that aren't trying to make good user-visible messages (all the
usual `logger.finest("sent an M, got two dollar signs and a comma")` kind of
thing) don't have any need to change.

[gmsg]:  http://docs.oracle.com/javase/7/docs/api/index.html?java/util/logging/LogRecord.html#getMessage()
[logrec]: http://docs.oracle.com/javase/7/docs/api/index.html?java/util/logging/LogRecord.html
[lgr]: http://docs.oracle.com/javase/7/docs/api/index.html?java/util/logging/Logger.html
[logr]: http://docs.oracle.com/javase/7/docs/api/index.html?java/util/logging/Logger.html#log(java.util.logging.LogRecord)

### Using familiar level names

The current implementation in PL/Java maps PostgreSQL severity levels onto
the (smaller set of) standard [`Level`][jlvl]s. This adds another bit of
cognitive load in the development process: I have to remember, for example,

```
SET log_min_messages TO DEBUG2;
SELECT javatest.logmessage('FINER', 'Hello world');
```

are talking about the same severity level, and it's an error to forget and
use the other name either place, and the mapping loses information—it's
not invertible.

A feature of the design of [`Level`][jlvl] is it can be subclassed, and
additional levels can be defined; the numeric values of the standard ones
are spaced widely apart to allow new ones between them, and the parser even
learns the names of new levels so they "just work". I propose defining the
PostgreSQL levels whose names do not already match [`Level`][jlvl] names,
with a possible relationship like this:

PostgreSQL|Java
--------:|-------
         |ALL
         |FINEST?
DEBUG5   |
         |FINEST?
DEBUG4   |
DEBUG3   |
         |FINER
DEBUG2   |
         |FINE
DEBUG1   |
         |CONFIG
LOG      |
COMMERROR|
INFO     |INFO
NOTICE   |
WARNING  |WARNING
         |SEVERE?
ERROR    |
         |SEVERE?
FATAL    |
PANIC    |
         |OFF

As you can see, I'm still considering arguments about where `FINEST` and
`SEVERE` should go.

The effect, again, is that code from elsewhere that only expects the usual
names from the standard library will work fine, code with more PostgreSQLy
origins can use those familiar names, and a developer or admin can set the
logging level using any of them, whichever seems more natural at the time.

### Could `pgjdbc` move to `java.util.logging` without disruption?

I think so. The class `org.postgresql.core.Logger` could be kept, and
simply delegate to other classes; the changes at points where log events
are read off the wire and exceptions are created should be fairly internal
and localized. I'd like to give it a shot.

I think the categorized exceptions in JDBC 4 are worth moving to, and
offer a much nicer way for client code to distinguish what kind of thing
went wrong, but changing _that_ might actually turn out to be what needs
the most coordination with client code. I would like to hope there isn't
much client code out there that has linked to `PSQLException` by name
(when it is only shown on "privateapi" javadocs), but I can only imagine
there is some.

## A place to pause

I have not managed to squeeze in every relevant thought here, but this is
already long and enough to elicit some discussion and questions, and if I
keep writing I will probably just be answering the wrong ones, so this
seems a good place to stop and listen.
