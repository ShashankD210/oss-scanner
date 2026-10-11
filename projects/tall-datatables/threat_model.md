# Threat model

## What this project does and where untrusted input enters

tall-datatables is a Laravel package (PHP 8.4, Laravel 12/13, Livewire 4, TallStackUI) that renders server side data
tables. A host application subclasses `src/DataTable.php`, sets the Eloquent model, optionally overrides
`getBuilder(Builder $builder)` to scope rows, and embeds the component in its pages. The package builds an Eloquent query
from the component state, renders rows through Blade (`resources/views/components/layouts/`), stores per user layouts and
saved filters in `datatable_user_settings` (`src/Models/DatatableUserSetting.php`), and exports xlsx, csv and json
(`src/Exports/`).

The attacker we care about is an authenticated user of the host application who can open a page containing a DataTable,
but who is limited by what the developer exposed and by the application's `getBuilder()` scope. Every public property and
every public method of a Livewire component is client controlled: the browser can send any property update and call any
public method with any arguments. Treat all of the following as untrusted:

* Public state of `src/DataTable.php` and its traits in `src/Traits/DataTables/`: enabled columns, user filters, text
  filters, sorting, search, paging, selection, grouping, aggregation, relation browsing, saved filters and layouts.
* Public methods on the same classes (filter loading and saving, sorting, grouping, exporting, relation loading, selection,
  kanban and row sorting, Echo listeners in `src/Traits/HasEloquentListeners.php`), including their arguments.
* The sub components `src/Components/DataTableOptions.php` and `src/Components/DataTableFilters.php`.
* HTTP routes in `routes/web.php`: the asset route (`src/Controllers/AssetController.php`) and the icon route
  (`src/Controllers/IconController.php`, `src/Helpers/Icon.php`).
* `src/Controllers/SearchController.php`, which the host application may route (config `search_route`). It takes a model
  class and query parameters from the request. When a host routes it, it is in scope.
* Stored data: record values rendered in cells, Scout search index content, and saved filters shared between users.

Client input reaches SQL mainly in `src/Traits/DataTables/BuildsQueries.php` and `src/Traits/DataTables/SupportsRelations.php`:
column names, relation paths in dot notation, operators and values end up in select, where, order by, group by, having,
`withCount`, `whereHas` and eager loading.

Responsibility split: the package knows nothing about tenants or permissions. The host application restricts rows in
`getBuilder()`, restricts columns and relations via `availableCols` and `availableRelations`, hides secrets via the model's
`$hidden`, and authorizes its own actions. If client input lets a user escape `getBuilder()`, reach columns or relations the
developer did not expose, read `$hidden` values, invoke model methods that are not relations, change `#[Locked]` state, or
act on records outside the scoped query, that is a package bug and in scope.

## Components that matter most / least

Most: `src/Traits/DataTables/BuildsQueries.php`, `SupportsRelations.php`, `SupportsGrouping.php`,
`SupportsAggregation.php`, `SupportsSelecting.php`, `SupportsExporting.php`, `StoresSettings.php`, `src/DataTable.php`,
`src/Controllers/SearchController.php`, `src/Helpers/SchemaInfo.php`, `src/Helpers/RelationFinder.php`, the Scout macros in
`src/DataTableServiceProvider.php`.

Rendering: the Blade layouts under `resources/views/components/layouts/` print formatted cell values as HTML, so output
safety depends on `src/Formatters/` and on how search highlight strings and appended attributes are passed through.

Least: `src/Commands/` (artisan only), `src/Casts/`, `src/Htmlables/`, `src/Helpers/DataTableTagCompiler.php` and
`DataTableBladeDirectives.php` (developer authored templates).

## How to exercise it

Tests use Pest 4 on Orchestra Testbench 11 with SQLite in memory (`tests/TestCase.php`). Fixture models live in
`tests/Fixtures/Models` and fixture tables in `tests/Fixtures/Livewire` (for example `PostDataTable`, `UserDataTable`,
`PostWithRelationsDataTable`, `SelectablePostDataTable`). `tests/Feature/SecurityTest.php` and
`tests/Feature/SharedFiltersTest.php` show the expected hardening.

```
cd /src
./vendor/bin/testbench package:test --testsuite=Unit,Feature
vendor/bin/pest tests/Feature/SecurityTest.php
```

`tests/Browser` needs Playwright and is not set up in this image. Write a proof of concept as a Feature test, for example
`Livewire::test(UserDataTable::class)->set('enabledCols', [...])->call('loadData')`, and assert on
`getDataForTesting()`, the query log (`DB::enableQueryLog()`) or the rendered HTML. SQLite lacks some MySQL functions, so
MySQL specific code paths need reasoning rather than a running test.

## How you rate severity

* Critical: SQL injection through a Livewire property, method argument or SearchController parameter that reads arbitrary
  tables or writes data; client input that runs a destructive or arbitrary model or query builder method; instantiation of
  arbitrary classes from client input that leads to code execution.
* High: SQL injection limited to the current table; escaping the `getBuilder()` scope to read or act on rows the host
  application did not expose (for example other tenants' records); reading `$hidden` attributes such as password hashes or
  tokens through any path (grouping, search highlights, exports, filter or sort oracles); selecting columns or relations
  outside a restrictive `availableCols` or `availableRelations`; changing `#[Locked]` properties from the client; stored
  XSS from record data or search index content rendered in cells when that data can come from lower privileged users.
* Medium: reading, modifying or deleting another user's saved filters or layouts; schema disclosure of models the table
  does not use; `javascript:` or open redirect links produced by built in formatters; exports that ignore the table's export
  setting.
* Low: CSV or xlsx formula injection in exports; LIKE wildcard abuse; resource exhaustion through paging, deep relation
  paths, grouping or aggregation; attribute injection in the icon route.

## Anything to leave alone

* `tests/`, `build/`, `stubs/`, `dist/`, `resources/js` and `resources/css` build tooling, `vite.config.js`,
  `.github/workflows`, and the Testbench skeleton under `vendor/orchestra`.
* Findings that need the developer to deliberately expose something dangerous: a `getBuilder()` that does not scope,
  `availableCols = ['*']` on a model with unhidden secrets, a custom formatter or `HtmlString` that returns unescaped HTML,
  a host route for `SearchController` without authentication. Report the package side only when the package defeats a
  restriction the developer did set.
* Self XSS, and anything that needs the attacker to already control the server, the configuration or the database.
* Missing authentication on the host application's own pages; the package assumes the host decides who may view a table.
* Vulnerabilities that live entirely in Laravel, Livewire, TallStackUI or PhpSpreadsheet, unless code in this package makes
  them reachable.
