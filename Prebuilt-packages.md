# Prebuilt PL/Java distributions

At present, the PL/Java project is reliant on downstream packagers to
produce prebuilt, installable PL/Java packages for various platforms. The
[official PL/Java releases][opljr] are offered in source form and take
only a few minutes to build with Apache Maven as described in the
[build instructions][bld].

[opljr]: https://github.com/tada/pljava/releases
[bld]: http://tada.github.io/pljava/build/build.html
[pljdv]: http://lists.pgfoundry.org/mailman/listinfo/pljava-dev

This wiki page will be updated to list known prebuilt PL/Java packages
and the platforms they are built for. As with any prebuilt distribution,
you should be acquainted with the policies and reputation of any supplier
of a prebuilt package. The PL/Java project has not directly built or verified
any package listed here.

## Known prebuilt packages available

### Debian/Ubuntu packages [on apt.postgresql.org][aptpgo]

    {1.5.2,"11.1 (Debian 11.1-1.pgdg+1)",11.0.1,Linux,amd64}`

PL/Java 1.5.2 packages available for PostgreSQL 11 back to 9.3 for Debian unstable/buster/stretch and Ubuntu cosmic/bionic/xenial, for amd64/i386/ppc64el. Dbgsym packages available. Includes `pljava-examples` jar with the [optional Saxon examples][osaxone] already built (download Saxon-HE 9.8.0.14 jar separately to use them).

[aptpgo]: http://apt.postgresql.org/pub/repos/apt/pool/main/p/postgresql-pljava/
[osaxone]: http://tada.github.io/pljava/examples/saxon.html

*Added 14 November 2018*

### Docker images

**Martin Bednar** has prepared [images][dockimg] of 64-bit PostgreSQL (9.5
and 9.4) with PL/Java 1.5.0 and Oracle Java 8 for use with [Docker][].

    {1.5.0,9.5.1,1.8.0_74,Linux,amd64}
    {1.5.0,9.4.6,1.8.0_74,Linux,amd64}

[Docker]: https://www.docker.com/
[dockimg]: https://hub.docker.com/r/xxbedy/postgres-pljava/tags/

*added 12 April 2016*

### Complete PostgreSQL distributions from BigSQL

[BigSQL][] provides native installers for Centos 6 and 7,
Ubuntu 12.04 and 14.04, OS X 10.9+, Windows 7+, and Windows Server 2008
and 2012. These distributions of PostgreSQL 9.5, 9.4, 9.3,
and 9.2 [include PL/Java 1.5.0][bsqplj].

[BigSQL]: http://www.bigsql.org/se/
[bsqplj]: http://www.bigsql.org/se/docs/proclang/proclang.jsp#pljava

*added 12 April 2016*

## To list a prebuilt package here

Please announce the availability of your package on
[the pljava-dev mailing list][pljdv], along with the output of
the third query below:

**Note: as of mid-May 2016, the `pljava-dev` mailing list is working again,
and should be used to announce packages. In case the mailing list does not
seem to work, then please [open an issue][ghissu].**

[ghissu]: https://github.com/tada/pljava/issues

`SELECT sqlj.install_jar(` _fileurl-to-built-pljava-examples-\*.jar_ `, 'ex', true);`  
`SELECT sqlj.set_classpath('javatest', 'ex');`  
```
SELECT array_agg(java_getsystemproperty(p)) FROM (values
('org.postgresql.pljava.version'),
('org.postgresql.version'),
('java.version'),
('os.name'),
('os.arch')
) AS props(p);
```

