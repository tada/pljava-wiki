# Tips for resolving build problems

Some typical issues encountered when building PL/Java can be listed here,
along with tips for resolving them.

## The tips that always apply

Please do carefully read the [build instructions][bld],
especially the "software prerequisites" section, and the "special topics"
section for any that apply to the platform where you are building.

[bld]: https://tada.github.io/pljava/build/build.html

Also be sure to review the "troubleshooting the build" section at the end
of the [build instructions page][bld].

If you review [the mailing list archive][pljdva] and the
[issues list][issues], you may find a report of a situation like your own.
(On the issues list, it is possible someone reported an issue, a solution
was found, and the issue was closed, so look at recent closed issues too.)

[pljdva]: http://lists.pgfoundry.org/pipermail/pljava-dev/
[issues]: https://github.com/tada/pljava/issues

## Failure shown for `pljava-so`

### Missing `-devel` prerequisite packages

The most common cause of reported failures building `pljava-so` is a
missing required file. Sometimes your distribution's packaging system will
have chosen to organize a prerequisite piece of software into more than
one package, for example, one that contains only library files, and another
with a name ending in `-dev` or `-devel` that contains the necessary `.h`
files. Some distributions take this further than others; see the "special
topics" section for Ubuntu for an example where even libraries built as
part of PostgreSQL itself are split up into multiple separate packages.

The solution is simple: look over the error messages from the `pljava-so`
section of the build output to find any that refer to a file that could not
be found. Usually it will be a `.h` file or a library (`.so`, `.dll`, `.dylib`,
etc.).

Find out the name of the package, according to the OS or package distribution
you are using, that contains the missing file, install that package,
and you have probably solved the whole problem.

**Further tip:** Finding the error message that really mattered is easier
if you follow the "troubleshooting the build" tip about the `-Pwnosign`
option, to cut down the number of other messages that do not matter, if
that option works on your platform.

## Still stuck?

Please describe the issue you are facing on
[the mailing list][pljdv].

[pljdv]: http://lists.pgfoundry.org/mailman/listinfo/pljava-dev
