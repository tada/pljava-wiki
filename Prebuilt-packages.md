# Prebuilt PL/Java distributions

At present, the PL/Java project is reliant on downstream packagers to
produce prebuilt, installable PL/Java jars for various platforms. The
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

## To list a prebuilt package here

Please announce the availability of your package on
[the pljava-dev mailing list][pljdv], along with the output of
the third query below:

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

