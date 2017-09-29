# NAME

DBICx::Sugar - Just some syntax sugar for DBIx::Class

# VERSION

version 0.0200

# SYNOPSIS

    use DBICx::Sugar qw(schema resultset rset);

    # all of the following are equivalent:

    $user = schema('default')->resultset('User')->find('bob');
    $user = schema->resultset('User')->find('bob');
    $user = resultset('User')->find('bob');
    $user = rset('User')->find('bob');

# DESCRIPTION

Just some syntax sugar for your DBIx::Class applications.
This was originally created to remove code duplication between
[Dancer::Plugin::DBIC](https://metacpan.org/pod/Dancer::Plugin::DBIC) and [Dancer2::Plugin::DBIC](https://metacpan.org/pod/Dancer2::Plugin::DBIC).

# CONFIGURATION

Configuration can be automatically parsed from a \`config.yaml\` or \`config.yml\`
file  in the current working directory, or it can be explicitly set with the
`config` function:

    DBICx::Sugar::config({ default => { dsn => ... } });

If you want the config to be autoloaded from a yaml config file, just make sure
to put your config data under a top level `dbicx_sugar` key.

## simple example

Here is a simple example. It defines one database named `default`:

    dbicx_sugar:
      default:
        dsn: dbi:SQLite:dbname=myapp.db
        schema_class: MyApp::Schema

## multiple schemas

In this example, there are 2 databases configured named `default` and `foo`:

    dbicx_sugar:
      default:
        dsn: dbi:SQLite:dbname=myapp.db
        schema_class: MyApp::Schema
      foo:
        dsn: dbi:Pg:dbname=foo
        schema_class: Foo::Schema
        user: bob
        password: secret
        options:
          RaiseError: 1
          PrintError: 1

Each database configured must at least have a dsn option.
The dsn option should be the [DBI](https://metacpan.org/pod/DBI) driver connection string.
All other options are optional.

If you only have one schema configured, or one of them is named
`default`, you can call `schema` without an argument to get the only
or `default` schema, respectively.

If a schema\_class option is not provided, then [DBIx::Class::Schema::Loader](https://metacpan.org/pod/DBIx::Class::Schema::Loader)
will be used to dynamically load the schema by introspecting the database
corresponding to the dsn value.
You need [DBIx::Class::Schema::Loader](https://metacpan.org/pod/DBIx::Class::Schema::Loader) installed for this to work.

WARNING: Dynamic loading is not recommended for production environments.
It is almost always better to provide a schema\_class option.

The schema\_class option should be the name of your [DBIx::Class::Schema](https://metacpan.org/pod/DBIx::Class::Schema) class.
See ["SCHEMA GENERATION"](#schema-generation)
Optionally, a database configuration may have user, password, and options
parameters as described in the documentation for `connect()` in [DBI](https://metacpan.org/pod/DBI).

## connect\_info

Alternatively, you may also declare your connection information inside an
array named `connect_info`:

    dbicx_sugar:
      default:
        schema_class: MyApp::Schema
        connect_info:
          - dbi:Pg:dbname=foo
          - bob
          - secret
          -
            RaiseError: 1
            PrintError: 1

## replicated

You can also add database read slaves to your configuration with the
`replicated` config option.
This will automatically make your read queries go to a slave and your write
queries go to the master.
Keep in mind that this will require additional dependencies:
[DBIx::Class::Optional::Dependencies#Storage::Replicated](https://metacpan.org/pod/DBIx::Class::Optional::Dependencies#Storage::Replicated)
See [DBIx::Class::Storage::DBI::Replicated](https://metacpan.org/pod/DBIx::Class::Storage::DBI::Replicated) for more details.
Here is an example configuration that adds two read slaves:

    dbicx_sugar:
      default:
        schema_class: MyApp::Schema
        dsn: dbi:Pg:dbname=master
        replicated:
          balancer_type: ::Random     # optional
          balancer_args:              # optional
              auto_validate_every: 5  # optional
              master_read_weight:1    # optional
          # pool_type and pool_args are also allowed and are also optional
          replicants:
            -
              - dbi:Pg:dbname=slave1
              - user1
              - password1
              -
                quote_names: 1
                pg_enable_utf8: 1
            -
              - dbi:Pg:dbname=slave2
              - user2
              - password2
              -
                quote_names: 1
                pg_enable_utf8: 1

## alias

Schema aliases allow you to reference the same underlying database by multiple
names.
For example:

    dbicx_sugar:
      default:
        dsn: dbi:Pg:dbname=master
        schema_class: MyApp::Schema
      slave1:
        alias: default

Now you can access the default schema with `schema()`, `schema('default')`,
or `schema('slave1')`.
This can come in handy if, for example, you have master/slave replication in
your production environment but only a single database in your development
environment.
You can continue to reference `schema('slave1')` in your code in both
environments by simply creating a schema alias in your development.yml config
file, as shown above.

# FUNCTIONS

## schema

    my $user = schema->resultset('User')->find('bob');

Returns a [DBIx::Class::Schema](https://metacpan.org/pod/DBIx::Class::Schema) object ready for you to use.
For performance, schema objects are cached in memory and are lazy loaded the
first time they are accessed.
If you have configured only one database, then you can simply call `schema`
with no arguments.
If you have configured multiple databases,
you can still call `schema` with no arguments if there is a database
named `default` in the configuration.
With no argument, the `default` schema is returned.
Otherwise, you **must** provide `schema()` with the name of the database:

    my $user = schema('foo')->resultset('User')->find('bob');

## resultset

This is a convenience method that will save you some typing.
Use this **only** when accessing the `default` schema.

    my $user = resultset('User')->find('bob');

is equivalent to:

    my $user = schema->resultset('User')->find('bob');

## rset

    my $user = rset('User')->find('bob');

This is simply an alias for `resultset`.

## get\_config

Returns the current configuration, like config does,
but does not look for a config file.

Use this for introspection, eg:

    my $dbix_sugar_is_configured = get_config ? 1 : 0 ;

## add\_schema\_to\_config

This function does not touch the existing config.
It can be used if some other part of your app
has configured DBICx::Sugar but did not know about
the part that uses an extra schema.

    add_schema_to_config('schema_name', { dsn => ... });

# SCHEMA GENERATION

Setting the schema\_class option and having proper DBIx::Class classes
is the recommended approach for performance and stability.
You can use the [dbicdump](https://metacpan.org/pod/dbicdump) command line tool provided by
[DBIx::Class::Schema::Loader](https://metacpan.org/pod/DBIx::Class::Schema::Loader) to help you.
For example, if your app were named Foo, then you could run the following
from the root of your project directory:

    dbicdump -o dump_directory=./lib Foo::Schema dbi:SQLite:/path/to/foo.db

For this example, your `schema_class` setting would be `'Foo::Schema'`.

# CONTRIBUTORS

- Henk van Oers <[https://github.com/hvoers](https://github.com/hvoers)>

# AUTHOR

Naveed Massjouni <naveed@vt.edu>

# COPYRIGHT AND LICENSE

This software is copyright (c) 2015 by Naveed Massjouni.

This is free software; you can redistribute it and/or modify it under
the same terms as the Perl 5 programming language system itself.
