## v0.1.0

The first release of iyi-mysql: crystal-mysql 0.18.0 ported to iyi, in iyi,
on iyi-db 0.3.0, with no Crystal code left.

* `mysql://` URIs over TCP or a Unix socket, the default database in the path
  or `database`, the collation in `encoding`, and `ssl-mode`, `ssl-ca`,
  `ssl-cert` and `ssl-key` (`Options`, `SSLOptions`).
* The protocol is written here over `std/socket`: the handshake, prepared
  statements in the binary protocol (COM_STMT_PREPARE, COM_STMT_EXECUTE) and
  unprepared ones in the text protocol (COM_QUERY), with `?` placeholders,
  `last_insert_id` and `rows_affected`.
* crystal-mysql's specs, and iyi-db's shared driver specs, ported and passing
  against MySQL 8.0.

### Beyond crystal-mysql

* **MySQL 8.0's default users connect.** crystal-mysql spoke
  `mysql_native_password` alone; the server's AuthSwitchRequest is answered
  now, and `caching_sha2_password` (its fast and its full authentication)
  and `sha256_password` are spoken. Without TLS the password goes under the
  server's RSA public key (RSAES-OAEP over `std/rsa`'s key and `std/big`).
* **TLS over `std/tls`, every certificate checked.** crystal-mysql checked
  none for `preferred` and `required`; here the chain and the host name are
  checked in every mode, against `ssl-ca` or the system's roots. `preferred`
  goes on unencrypted when the server does not take TLS or its certificate
  is not trusted, where crystal-mysql required it. A Unix socket is never
  encrypted. Client certificates are sent from `ssl-cert` and `ssl-key`.
* **Unsigned columns read as unsigned** (`UInt8` ... `UInt64`), where
  crystal-mysql read `255` in a `TINYINT UNSIGNED` as `-1`; unsigned
  arguments bind as unsigned.
* **Every column type reads**: `JSON` as `String`, `YEAR` as `Int16`, `BIT`
  and `GEOMETRY` as `Bytes`, `ENUM` and `SET` as `String`, where
  crystal-mysql raised for `JSON`, `YEAR` and `BIT`. A binary character set
  reads as `Bytes` whatever the type, so `BINARY` and `VARBINARY` do too.
  `TIME` reads and binds negative values and values past 24 hours.
* **Prepared statements are closed on the server** (COM_STMT_CLOSE) when
  they are closed, where crystal-mysql left them until the connection
  closed.
* **Payloads of 16 MiB and more** are read and written across packets.
* A server error is a `PacketError` with its `code` and `sql_state`; the
  codes that mean the session is gone, and a socket failure, are iyi-db's
  `ConnectionLost`, so the pool reconnects.
* `MySqlConnection#connection_id` is the server's id for the session.

### What iyi's rules changed

* Errors are values: `PacketError`, `ConnectionError` and `StreamError` are
  `DBError`s, and an unknown `ssl-mode` or `encoding` is an `ArgumentError`.
* The driver classes are `MySqlDriver`, `MySqlConnection`, `MySqlStatement`,
  `MySqlUnpreparedStatement`, `MySqlResultSet` and `MySqlTextResultSet` in
  `mysql/core`.
* A `UUID` binds as `UUIDValue.new(uuid)` (`DBValue` is closed);
  `read(UUID)` reads one from a binary column as before.
* `exec` of a statement that has rows reads them and answers zeros, and
  `query` of one that has none answers no rows, where crystal-mysql's casts
  raised and left the session out of step.
* A zero date (`0000-00-00`) answers a `DBError`, as iyi's `Time` has no
  year 0.

### Known issue

* In a debug build of iyi 0.18.0 a TLS handshake run as a task can overrun
  the task's 256 KiB stack and end in a memory fault or a certificate read as
  untrusted ([iyi#130](https://github.com/iyilang/iyi/issues/130)). A
  `--release` build, or the handshake on the main fiber, is not affected.

crystal-mysql's history before the port is in its
[CHANGELOG](https://github.com/crystal-lang/crystal-mysql/blob/master/CHANGELOG.md).
