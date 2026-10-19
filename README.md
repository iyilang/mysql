# iyi-mysql

A native MySQL driver for [iyi](https://iyi.dev): a port of
[crystal-mysql](https://github.com/crystal-lang/crystal-mysql) 0.18.0, written
in iyi with no Crystal code and no Crystal standard library underneath. It is a
driver for [iyi-db](https://github.com/iyilang/db), the common database
interface, and speaks the MySQL client/server protocol itself.

It keeps crystal-mysql's shape - `mysql://` URIs, prepared statements in the
binary protocol and unprepared ones in the text protocol, `?` placeholders,
TCP and Unix sockets, TLS with `ssl-mode`, collations by name - and changes
what iyi's rules require: errors are values, types are closed, and tasks are
structured. It also connects to what crystal-mysql could not: MySQL 8.0's
`caching_sha2_password` users, and `sha256_password` ones.

## Requirements

iyi 0.18.0 or later, and iyi-db 0.3.0 (required by this package's `iyi.mod`).
Tested against MySQL 8.0.

## Installation

Add it to a project under the short name `mysql`, and iyi-db under `db`, whose
`Db.open` a program calls:

```sh
iyi get github.com/iyilang/mysql --as mysql
iyi get github.com/iyilang/db --as db
```

Files then `import mysql` for the root module - `Mysql::VERSION` - and its
other modules as `mysql/mysql/...`: the short name, then the module's path in
the package. Importing `mysql` (or `mysql/mysql/core`) registers the driver for
`mysql://`. A package required only by another is not named in a program's
own imports, so `import db` needs the second line.

## Usage

```iyi
module app

import db
import db/db/errors::{DBError}
import mysql

def contacts : Nil | DBError
  # connect to localhost mysql test db
  Db.open("mysql://root@localhost/test") do |db|
    db.exec("drop table if exists contacts")!
    db.exec("create table contacts (name varchar(30), age int)")!
    db.exec("insert into contacts values (?, ?)", "John Doe", 30)!
    db.exec("insert into contacts values (?, ?)", args: ["Sarah", 33])!

    puts "max age:"
    puts db.scalar("select max(age) from contacts")! # => 33

    puts "contacts:"
    db.query("select name, age from contacts order by age desc") do |rs|
      puts "#{rs.column_name(0)} (#{rs.column_name(1)})"
      # => name (age)
      rs.each do
        puts "#{rs.read(String)!} (#{rs.read(Int32)!})"
        # => Sarah (33)
        # => John Doe (30)
      end!
    end!
  end
end

case contacts
in Nil     then puts "done"
in DBError then puts "failed: #{it.message}"
end
```

A server error is a `PacketError` (module `mysql/errors`) with the server's
`code` and `sql_state`: `1146` and `"42S02"` for a table that does not exist.
A connection that cannot be made is iyi-db's `ConnectionRefused`, whose `cause`
says why - a `PacketError` for a refused password (`1045`), a
`ConnectionError` for a socket or TLS failure.

### Connection URI

The connection string has the following syntax:

```
mysql://[user[:[password]]@]host[:port][/schema][?param1=value1&param2=value2]
```

#### Transport

The driver supports tcp connection or unix sockets

- `mysql://localhost` will connect using tcp and the default MySQL port 3306.
- `mysql://localhost:8088` will connect using tcp using port 8088.
- `mysql:///path/to/other.sock` will connect using unix socket `/path/to/other.sock`.

Any of the above can be used with `user@` or `user:password@` to pass
credentials. A host is a name, resolved by iyi's own resolver, or an address.

#### Default database

A `database` query string will specify the default database. Connection
strings with a host can also use the first path component to specify the
default database. Query string takes precedence because it's more explicit.

- `mysql://localhost/mydb`
- `mysql://localhost:3306/mydb`
- `mysql://localhost:3306?database=mydb`
- `mysql:///path/to/other.sock?database=mydb`

#### Secure connections (SSL/TLS)

Over TCP a session is encrypted with iyi's `std/tls` as `ssl-mode` says;
a Unix socket is never encrypted.

- `ssl-mode`: `disabled`; `preferred`, the default, when the server takes TLS
  and its certificate is trusted, unencrypted otherwise; `required`,
  `verify_ca` and `verify_identity` always. Any case, with `-`, `_` or nothing
  between the words (`VERIFY_CA`, `verify-ca`, `verifyca`).
- `ssl-key`: Path to the client key.
- `ssl-cert`: Path to the client certificate.
- `ssl-ca`: Path to the CA certificate.

```iyi
Db.open("mysql://user:pass@db.example.com/mydb?ssl-mode=verify_identity&ssl-ca=/etc/ssl/mysql-ca.pem")
```

`std/tls` checks every server's certificate - its chain to a root, and the
host name - and cannot be told not to, so this driver checks it whatever the
`ssl-mode`, where crystal-mysql checked nothing for `preferred` and `required`.
The roots are the ones in `ssl-ca`, or the system's without it. The
certificate MySQL generates for itself names no host, and so is trusted for
none: a server that is to be reached over TLS needs a certificate for its host
name, and its CA in `ssl-ca` unless the system trusts it. With `preferred`, a
server whose certificate is not trusted is reached unencrypted, after a
handshake that fails.

A server that asks for a client certificate is sent the one in `ssl-cert` and
`ssl-key`: PEM, the key Ed25519, P-256 or RSA, unencrypted. TLS 1.3 is spoken,
and 1.2 with a server that has nothing newer.

In iyi 0.18.0, a handshake run as a task in a debug build can overrun the
task's 256 KiB stack - the certificate's checks take most of it in a debug
build, with an RSA certificate as with an ECDSA one - and the program dies of a
memory fault or reads a certificate as untrusted
([iyi#130](https://github.com/iyilang/iyi/issues/130)). A pool opens its
connections in the task that checks them out. A `--release` build, or the
handshake on the main fiber, is not affected.

#### Authentication

The server's authentication plugin for the user is answered, switching to it
when the server asks:

- `mysql_native_password`.
- `caching_sha2_password`, MySQL 8.0's default. When the server has no cached
  entry for the user - the first connection after it starts, or after `FLUSH
  PRIVILEGES` - it asks for the password whole: sent as it is over TLS or a
  Unix socket, and otherwise encrypted under the server's RSA public key
  (RSAES-OAEP), which the server sends on request.
- `sha256_password`, the same way.

#### Other query params

- `encoding`: The collation & charset (character set) to use during the
  connection. If empty or not defined, it will be set to `utf8_general_ci`,
  MySQL's three-byte UTF-8 (`utf8mb3`); `utf8mb4_general_ci`, or
  `utf8mb4_0900_ai_ci` on MySQL 8.0, stores every character. The list of
  available collations is `COLLATIONS_IDS_BY_NAME` in `mysql/collations`; a
  name not in it is an `ArgumentError`.

And iyi-db's pool and connection options: `initial_pool_size`,
`max_pool_size`, `max_idle_pool_size`, `checkout_timeout`, `retry_attempts`,
`retry_delay`, `prepared_statements`, `prepared_statements_cache`.

## Supported datatypes

| MySQL | read as |
|---|---|
| tinyint, smallint, mediumint, int, bigint | `Int8`, `Int16`, `Int32`, `Int32`, `Int64`; `UInt8`, ... `UInt64` when `UNSIGNED` |
| year | `Int16` (`UInt16` when `UNSIGNED`, as MySQL sends it) |
| float, double | `Float32`, `Float64` |
| decimal | `Float64` |
| datetime, timestamp, date | `Time` (UTC) |
| time | `Span`, negative ones included |
| char, varchar, text, enum, set | `String` |
| binary, varbinary, blob | `Bytes` |
| json | `String` |
| bit, geometry | `Bytes` |
| bool (`tinyint(1)`), and any integer | `read(Bool)`: not zero |
| binary(16), blob holding 16 bytes | `read(UUID)`, `read(UUIDValue)` |

Whether a string column reads as `String` or `Bytes` is its character set's:
binary is `Bytes`. A `Time` is written and read as its wall clock in UTC, as
crystal-mysql's are: a `TIMESTAMP` column is stored in the session's
`time_zone`, so a session whose zone is not UTC (`SET time_zone = '+00:00'`)
stores the instant shifted, and reads it back shifted the same way. A zero
date (`0000-00-00`) answers a `DBError`.

Values bound as arguments are iyi-db's `DBValue`s: `Nil`, `Bool` (as 1 or 0),
the signed and unsigned integers, `Float32`, `Float64`, `String`, `Char`,
`Bytes`, `Time` (to the microsecond), `Span` (at most 34 days, MySQL `TIME`'s
range) and `UUIDValue` (its 16 bytes). Another value is refused with a
`DBError`, before anything is sent. Unprepared statements take no arguments.

## Modules

| module | what it holds |
|---|---|
| `mysql` (the root module) | `VERSION`; re-exports `mysql/core` |
| `mysql/core` | `MySqlDriver`, `MySqlConnection`, `MySqlStatement`, `MySqlUnpreparedStatement`, `MySqlResultSet`, `MySqlTextResultSet`, `MySqlConnectionBuilder`; registers the driver |
| `mysql/options` | `Options`, `SSLOptions`, `SSLMode`: what a URI says |
| `mysql/collations` | `COLLATIONS_IDS_BY_NAME`, `default_collation`, `id_for_collation` |
| `mysql/types` | `Type`, `ColumnSpec`, `UUIDValue`, and how each type is read and bound |
| `mysql/session` | `Session`, the protocol session: handshake, authentication, commands |
| `mysql/auth` | the authentication plugins' answers, and RSAES-OAEP |
| `mysql/ssl` | the `TlsConfig` an `SSLOptions` asks for |
| `mysql/packet`, `mysql/stream` | packets, and the buffered socket or TLS session |
| `mysql/errors` | `ConnectionError`, `StreamError`, `PacketError` |

## What is different from crystal-mysql

| crystal-mysql | iyi-mysql | why |
|---|---|---|
| raises `MySql::Connection::PacketError`, `DB::ConnectionLost`, `IO::Error`, `Exception` | answers them as `DBError`s: `PacketError` (with `code` and `sql_state`), `StreamError`; a statement answers a socket failure, and the server's "gone away" codes, as iyi-db's `ConnectionLost` | errors are values (SPEC.md III.1) |
| `MySql::Connection`, `MySql::Statement`, `MySql::ResultSet`, `MySql::Driver` | `MySqlConnection`, `MySqlStatement`, `MySqlResultSet`, `MySqlDriver` in one module | the module imports iyi-db's `Connection`, ... by those names, and they refer to one another (R-1) |
| `Connection::Options.from_uri`, `SSLOptions.from_params` raise | answer `Options \| DBError`, `SSLOptions \| DBError`; an unknown `encoding` is an `ArgumentError` where crystal-mysql sent collation 0 | a constructor has no error to answer |
| `mysql_native_password` only | `caching_sha2_password` and `sha256_password` too, and the server's AuthSwitchRequest answered | MySQL 8.0 creates users with `caching_sha2_password` |
| `ssl-mode=preferred` behaves as `required`, checking no certificate; `verify_ca` checks no host name | every certificate is checked, chain and host name; `preferred` goes on unencrypted when the server's is not trusted | `std/tls` has no way to turn the checks off |
| TLS over a Unix socket for `required` and stricter | a Unix socket is never encrypted | the server takes a socket as secure |
| an `UNSIGNED` column reads as the signed type (255 as -1) | as the unsigned type | so it reads what it holds |
| `BINARY`/`VARBINARY` read as `String`; `JSON`, `YEAR`, `BIT` raise | `Bytes`; `String`, `Int16`, `Bytes` | a binary character set is bytes; every type the server sends reads |
| a prepared statement stays on the server until the connection closes | closed there (COM_STMT_CLOSE) when the statement is closed | the server holds at most `max_prepared_stmt_count` |
| a packet of 16 MiB or more cannot be read or written | read and written across packets | the protocol cuts it so |
| `exec` of a statement with rows raises; `query` of one without raises | `exec` reads the rows and answers zeros; `query` answers no rows | the session stays in step |
| a UUID binds as its bytes | `UUIDValue.new(uuid)` binds; `read(UUID)` reads | `DBValue` is closed (iyi-db) |
| arguments not matching the statement's `?`s are sent | refused before sending | the server cannot read the packet |

## Development

```sh
DATABASE_URL=mysql://root:password@127.0.0.1?ssl-mode=disabled iyi test mysql
```

The tests are crystal-mysql's specs, ported, beside the modules they test, with
iyi-db's shared driver specs (`mysql/driver_specs_test.iyi`), and tests of what
this port adds: the authentication plugins (`mysql/auth_test.iyi`), the types
(`mysql/types_test.iyi`), packets across frames (`mysql/protocol_test.iyi`) and
TLS (`mysql/ssl_test.iyi`). They read `DATABASE_URL`, `mysql://root@localhost`
by default, whose user creates and drops the database `crystal_mysql_test` and
test users. The TLS tests, and the driver's "connect with ssl", run when
`SSL_CA` names the CA of a server certificate for the test host, and the client
certificate test when `SSL_CLIENT_CERT` and `SSL_CLIENT_KEY` name a
certificate for `CN=iyi_client` and its key; `.github/workflows/ci.yml` makes
them. Run the suite over TLS from `--release` builds (see TLS above):

```sh
export DATABASE_URL="mysql://root:password@127.0.0.1?ssl-mode=verify_identity&ssl-ca=$SSL_CA"
for test in mysql/*_test.iyi; do iyi build --release "$test" -o bin/test && bin/test; done
```

## Contributors

crystal-mysql is the work of [Juan Wajnerman](https://github.com/waj), [Brian
J. Cardiff](https://github.com/bcardiff) and its
[contributors](https://github.com/crystal-lang/crystal-mysql/graphs/contributors);
this port keeps its licence (LICENSE).
