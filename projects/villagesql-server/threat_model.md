# Threat model

## What this project does and where untrusted input enters
VillageSQL Server is a tracking fork of MySQL 8.4 LTS that adds the VillageSQL Extension Framework (VEF):
extension bundles (`.veb` files) that add custom data types and custom functions to the server. Clients connect
over the MySQL classic protocol (TCP 3306, Unix socket) and the X Protocol (TCP 33060). The project is alpha.

Untrusted input, most to least important:
- Pre-authentication input: the connection handshake and authentication exchange, including any VillageSQL
  changes in `sql/auth/sql_authentication.cc`.
- Authenticated users with ordinary privileges (for example `SELECT`/`INSERT` on one table). Everything they can
  send is in scope, in particular:
  - values for custom-type columns (string to custom-type encoding and decoding, implicit casts, comparisons,
    hashing, sorting, indexing), arriving through `INSERT`, `UPDATE`, `LOAD DATA`, `LOAD XML`, prepared
    statements, stored procedure parameters, triggers, views, window and aggregate functions;
  - custom-type parameters written in DDL (for example `CREATE TABLE t (c SOMETYPE(...))`);
  - calls to extension functions (VDF), including argument and result handling on the server side.
- Escaping the privilege model is a vulnerability: running `INSTALL`/`UNINSTALL`/`ALTER EXTENSION` without
  `EXTENSION_ADMIN` or `SUPER`, changing `vsql_allow_preview_extensions` without the needed privilege, reading or
  writing the `villagesql.*` system tables from SQL (they are meant to be reachable only by the server's
  data-dictionary code), or reading data in tables the user has no grant on.
- Users with `EXTENSION_ADMIN`: in scope, but the attacker is already highly privileged. Such a user may load any
  bundle that is already in `veb_dir`, and that bundle runs as native code inside `mysqld`. They are *not* trusted
  to reach files outside `veb_dir`. Loading code from elsewhere, or reading or writing files outside `veb_dir` and
  `<datadir>/.veb_expansion_cache/`, through an extension name, a `VERSION` string, or the contents of a bundle,
  is a vulnerability.
- Clone (`plugin/clone`) and replication peers: in scope, with the high privileges they require.

Trusted:
- Extension code. A loaded extension is native code `dlopen`ed into `mysqld`, at the same trust level as
  `INSTALL PLUGIN`. Do not report that a malicious extension can compromise the server. The server must still
  honour its side of the VEF ABI (`villagesql/sdk/include/villagesql/abi/types.h`). For example, results may report
  a length larger than the buffer they were given (snprintf-style), and the server must not read or write past its
  own buffers when an extension behaves exactly as the ABI allows. Server bugs of that kind are in scope.
- Files in `veb_dir` (set only on the command line or in the config file), the config file, the command line,
  the data directory and anything else on the host's filesystem.

## Components that matter most / least
- Most: VillageSQL code under `villagesql/` (`veb/`, `vdf/`, `types/`, `schema/`, `services/`, `sql/`,
  `system_views/`), and the VillageSQL hooks it adds elsewhere: `sql/sql_yacc.yy`, `sql/sql_table.cc`,
  `sql/item*.cc`, `sql/field.cc`, `sql/mysqld.cc`, `sql/auth/`, `sql/dd/impl/dictionary_impl.cc`,
  `storage/innobase/villagesql/`, `storage/innobase/handler/`, `storage/innobase/rem/rem0cmp.cc`, and
  `plugin/clone/`. Use `git grep -il villagesql` to find the rest.
- Less: unmodified upstream MySQL 8.4 code. It is in scope, but say in the report whether the bug also exists in
  Oracle MySQL 8.4, so it can be handled with upstream.
- Out of scope: third-party code under `extra/` (libarchive, curl, rapidjson, protobuf, and so on) unless
  VillageSQL calls it incorrectly; MySQL Router (not built); the deliberately misbehaving extensions in
  `villagesql/test-extensions/`; the example extensions in `villagesql/examples/` except as a harness; build and
  CI scripts. Extensions that live in other repositories (vsql-uuid, vsql-ai, vsql-http, ...) are not part of
  this project.

## How to exercise it
- `/build` is a Debug build with ASan and UBSan. Binaries are in `/build/bin`. Rebuild after a patch with
  `cmake --build /build --target mysqld` (add other targets as needed; building everything is slow on two CPUs).
- Start a server on the pre-initialized data directory:
  `/build/bin/mysqld --no-defaults --user=root --basedir=/build --datadir=/data --socket=/tmp/mysql.sock &`
  then connect with `/build/bin/mysql -uroot --socket=/tmp/mysql.sock`.
- Bundled extensions are built into `/build/lib/veb` (the default `veb_dir`). Try them with
  `INSTALL EXTENSION vsql_complex;` (custom type `COMPLEX` and `complex_*` functions) or `vsql_simple`, and
  inspect them with `SELECT * FROM INFORMATION_SCHEMA.EXTENSIONS;`.
- Integration tests are MTR tests under `mysql-test/suite/villagesql/<area>/`. Run them from `/build/mysql-test`:
  `./mysql-test-run.pl --parallel=2 villagesql/select.<test>` or `--suite=villagesql/<area>`. MTR copies the
  built `.veb` files into each test server. Shared helpers are in `mysql-test/include/villagesql/`. Wrap long runs
  in `timeout`.
- Unit tests (`unittest/gunit/villagesql/`) are not built in this image. To use them, reconfigure with
  `cmake -S /src -B /build -DWITH_UNIT_TESTS=ON`, build the `villagesql-unit-tests` target, then run
  `cd /build && ctest -L villagesql`. This is slow on two CPUs; prefer MTR tests.
- `Docs/` and `villagesql/sdk/README.md` describe the SDK, ABI versions and conventions.

## How you rate severity
Score with CVSS 3.1 and give the vector.

- AV: `N` for anything reachable over the client protocol.
- PR: `N` for pre-authentication bugs. `L` for a user with ordinary table privileges. `H` for `EXTENSION_ADMIN`,
  `SUPER`, clone or replication privileges.
- A crash of `mysqld` is `A:H`. A bounded out-of-bounds read is `C:L` unless it discloses other users' data.
  Arbitrary memory write or code execution as the `mysqld` user is `C:H/I:H/A:H`.
- Authentication bypass, or any pre-authentication memory corruption, is critical.
- Memory corruption that an ordinary user can reach and plausibly control is at least high.
- Bypassing a grant to read or change another user's data, or gaining `EXTENSION_ADMIN`, is high.
- Unbounded CPU or memory use from a query of reasonable size is medium. Slow or memory-hungry queries that scale
  with the data or input size are not vulnerabilities.

This image is a Debug build, so `assert` and `DBUG_ASSERT` checks are live. An assertion failure on its own is
low. Say what happens in a release build (`-DCMAKE_BUILD_TYPE=RelWithDebInfo`, no `WITH_DEBUG`): if the same input
causes memory corruption or a crash there, rate that outcome. A sanitizer report alone does not set severity;
explain the reachable impact.

## Anything to leave alone
- Configurations that disable security: `--skip-grant-tables`, `--initialize-insecure` accounts with no
  password, and debug-only switches such as `SET debug = ...` and `skip_dd_table_access_check`.
- Behaviour of a malicious or buggy extension, as described under Trusted above.
- Known limitations listed in `README.md` (no Windows support, no `SUM`/`AVG` on custom types, and so on).

## Reports and patches
- Target the `main` branch.
- Give the reproducer as an MTR test (`.test` plus expected `.result`) or as a SQL script for the `mysql` client,
  starting from a fresh server.
- Patches should follow `AGENTS.md`: `//` comments, `TODO(villagesql): ...` style, MySQL file-system wrappers, and
  a new or updated MTR test.
- Findings go privately to `security@villagesql.com`. Do not put suspected vulnerabilities or reproducers in public
  issues or pull requests.
