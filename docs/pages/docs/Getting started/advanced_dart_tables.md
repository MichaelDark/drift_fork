---
data:
  title: "Dart tables"
  description: "Further information on defining tables in Dart. This page describes advanced features like constraints, nullability, references and views"
  weight: 150
template: layouts/docs/single
---

{% block "blocks/pageinfo" %}
__Prefer sql?__ If you prefer, you can also declare tables via `CREATE TABLE` statements.
Drift's sql analyzer will generate matching Dart code. [Details]({{ "starting_with_sql.md" | pageUrl }}).
{% endblock %}

{% assign snippets = 'package:drift_docs/snippets/tables/advanced.dart.excerpt.json' | readString | json_decode %}

As shown in the [getting started guide]({{ "index.md" | pageUrl }}), sql tables can be written in Dart:
```dart
class Todos extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get title => text().withLength(min: 6, max: 32)();
  TextColumn get content => text().named('body')();
  IntColumn get category => integer().nullable()();
}
```

In this article, we'll cover some advanced features of this syntax.

## Names

By default, drift uses the `snake_case` name of the Dart getter in the database. For instance, the
table
```dart
class EnabledCategories extends Table {
    IntColumn get parentCategory => integer()();
    // ..
}
```

Would be generated as `CREATE TABLE enabled_categories (parent_category INTEGER NOT NULL)`.

To override the table name, simply override the `tableName` getter. An explicit name for
columns can be provided with the `named` method:
```dart
class EnabledCategories extends Table {
    String get tableName => 'categories';

    IntColumn get parentCategory => integer().named('parent')();
}
```

The updated class would be generated as `CREATE TABLE categories (parent INTEGER NOT NULL)`.

To update the name of a column when serializing data to json, annotate the getter with
[`@JsonKey`](https://pub.dev/documentation/drift/latest/drift/JsonKey-class.html).

You can change the name of the generated data class too. By default, drift will stip a trailing
`s` from the table name (so a `Users` table would have a `User` data class).
That doesn't work in all cases though. With the `EnabledCategories` class from above, we'd get
a `EnabledCategorie` data class. In those cases, you can use the [`@DataClassName`](https://pub.dev/documentation/drift/latest/drift/DataClassName-class.html)
annotation to set the desired name.

## Nullability

By default, columns may not contain null values. When you forgot to set a value in an insert,
an exception will be thrown. When using sql, drift also warns about that at compile time.

If you do want to make a column nullable, just use `nullable()`:
```dart
class Items {
    IntColumn get category => integer().nullable()();
    // ...
}
```

## Checks

If you know that a column (or a row) may only contain certain values, you can use a `CHECK` constraint
in SQL to enforce custom constraints on data.

In Dart, the `check` method on the column builder adds a check constraint to the generated column:

```dart
  // sqlite3 will enforce that this column only contains timestamps happening after (the beginning of) 1950.
  DateTimeColumn get creationTime => dateTime()
      .check(creationTime.isBiggerThan(Constant(DateTime(1950))))
      .withDefault(currentDateAndTime)();
```

Note that these `CHECK` constraints are part of the `CREATE TABLE` statement.
If you want to change or remove a `check` constraint, write a [schema migration]({{ '../Advanced Features/migrations.md#changing-column-constraints' | pageUrl }}) to re-create the table without the constraint.

## Default values

You can set a default value for a column. When not explicitly set, the default value will
be used when inserting a new row. To set a constant default value, use `withDefault`:

```dart
class Preferences extends Table {
  TextColumn get name => text()();
  BoolColumn get enabled => boolean().withDefault(const Constant(false))();
}
```

When you later use `into(preferences).insert(PreferencesCompanion.forInsert(name: 'foo'));`, the new
row will have its `enabled` column set to false (and not to null, as it normally would).
Note that columns with a default value (either through `autoIncrement` or by using a default), are
still marked as `@required` in generated data classes. This is because they are meant to represent a
full row, and every row will have those values. Use companions when representing partial rows, like
for inserts or updates.

Of course, constants can only be used for static values. But what if you want to generate a dynamic
default value for each column? For that, you can use `clientDefault`. It takes a function returning
the desired default value. The function will be called for each insert. For instance, here's an
example generating a random Uuid using the `uuid` package:
```dart
final _uuid = Uuid();

class Users extends Table {
    TextColumn get id => text().clientDefault(() => _uuid.v4())();
    // ...
}
```

Don't know when to use which? Prefer to use `withDefault` when the default value is constant, or something
simple like `currentDate`. For more complicated values, like a randomly generated id, you need to use
`clientDefault`. Internally, `withDefault` writes the default value into the `CREATE TABLE` statement. This
can be more efficient, but doesn't support dynamic values.

## Primary keys

If your table has an `IntColumn` with an `autoIncrement()` constraint, drift recognizes that as the default
primary key. If you want to specify a custom primary key for your table, you can override the `primaryKey`
getter in your table:

```dart
class GroupMemberships extends Table {
  IntColumn get group => integer()();
  IntColumn get user => integer()();

  @override
  Set<Column> get primaryKey => {group, user};
}
```

Note that the primary key must essentially be constant so that the generator can recognize it. That means:

- it must be defined with the `=>` syntax, function bodies aren't supported
- it must return a set literal without collection elements like `if`, `for` or spread operators

## Unique Constraints

Starting from version 1.6.0, `UNIQUE` SQL constraints can be defined on Dart tables too.
A unique constraint contains one or more columns. The combination of all columns in a constraint
must be unique in the table, or the database will report an error on inserts.

With drift, a unique constraint can be added to a single column by marking it as `.unique()` in
the column builder.
A unique set spanning multiple columns can be added by overriding the `uniqueKeys` getter in the
`Table` class:

{% include "blocks/snippet" snippets = snippets name = 'unique' %}

## Supported column types

Drift supports a variety of column types out of the box. You can store custom classes in columns by using
[type converters]({{ "../Advanced Features/type_converters.md" | pageUrl }}).

| Dart type    | Column        | Corresponding SQLite type                           |
|--------------|---------------|-----------------------------------------------------|
| `int`        | `integer()`   | `INTEGER`                                           |
| `BigInt`     | `int64()`     | `INTEGER` (useful for large values on the web)      |
| `double`     | `real()`      | `REAL`                                              |
| `boolean`    | `boolean()`   | `INTEGER`, which a `CHECK` to only allow `0` or `1` |
| `String`     | `text()`      | `TEXT`                                              |
| `DateTime`   | `dateTime()`  | `INTEGER` (default) or `TEXT` depending on [options](#datetime-options)               |
| `Uint8List`  | `blob()`      | `BLOB`                                              |
| `Enum`       | `intEnum()`   | `INTEGER` (more information available [here]({{ "../Advanced Features/type_converters.md#implicit-enum-converters" | pageUrl }})). |
| `Enum`       | `textEnum()`   | `TEXT` (more information available [here]({{ "../Advanced Features/type_converters.md#implicit-enum-converters" | pageUrl }})). |

Note that the mapping for `boolean`, `dateTime` and type converters only applies when storing records in
the database.
They don't affect JSON serialization at all. For instance, `boolean` values are expected as `true` or `false`
in the `fromJson` factory, even though they would be saved as `0` or `1` in the database.
If you want a custom mapping for JSON, you need to provide your own [`ValueSerializer`](https://pub.dev/documentation/drift/latest/drift/ValueSerializer-class.html).

### `BigInt` support

Drift supports the `int64()` column builder to indicate that a column stores
large integers and should be mapped to Dart as a `BigInt`.

This is mainly useful for Dart apps compiled to JavaScript, where an `int`
really is a `double` that can't store large integers without loosing information.
Here, representing integers as `BigInt` (and passing those to the underlying
database implementation) ensures that you can store large intergers without any
loss of precision.
Be aware that `BigInt`s have a higher overhead than `int`s, so we recommend using
`int64()` only for columns where this is necessary:

{% block "blocks/alert" title="You might not need this!" color="info" %}
In sqlite3, an `INTEGER` column is stored as a 64-bit integer.
For apps running in the Dart VM (e.g. on everything except for the web), the `int`
type in Dart is the _perfect_ match for that since it's also a 64-bit int.
For those apps, we recommend using the regular `integer()` column builder.

Essentially, you should use `int64()` if both of these are true:

- you're building an app that needs to work on the web, _and_
- the column in question may store values larger than 2<sup>52</sup>.

In all other cases, using a regular `integer()` column is more efficient.
{% endblock %}

Here are some more pointers on using `BigInt`s in drift:

- Since an `integer()` and a `int64()` is the same column in sqlite3, you can
  switch between the two without writing a schema migration.
- In addition to large columns, it may also be that you have a complex expression
  in a select query that would be better represented as a `BigInt`. You can use
  `dartCast()` for this: For an expression
  `(table.columnA * table.columnB).dartCast<BigInt>()`, drift will report the
  resulting value as a `BigInt` even if `columnA` and `columnB` were defined
  as regular integers.
- `BigInt`s are not currently supported by `moor_flutter` and `drift_sqflite`.
- To use `BigInt` support on a `WebDatabase`, set the `readIntsAsBigInt: true`
  flag when instantiating it.
- Both `NativeDatabase` and `WasmDatabase` have builtin support for bigints.

### `DateTime` options

Drift supports two approaches of storing `DateTime` values in SQL:

1. __As unix timestamp__ (the default): In this mode, drift stores date time
   values as an SQL `INTEGER` containing the unix timestamp (in seconds).
   When date times are mapped from SQL back to Dart, drift always returns a
   non-UTC value. So even when UTC date times are stored, this information is
   lost when retrieving rows.
2. __As ISO 8601 string__: In this mode, datetime values are stored in a
   textual format based on `DateTime.toIso8601String()`: UTC values are stored
   unchanged (e.g. `2022-07-25 09:28:42.015Z`), while local values have their
   UTC offset appended (e.g. `2022-07-25T11:28:42.015 +02:00`).
   Most of sqlite3's date and time functions operate on UTC values, but parsing
   datetimes in SQL respects the UTC offset added to the value.

   When reading values back from the database, drift will use `DateTime.parse`
   as following:
    - If the textual value ends with `Z`, drift will use `DateTime.parse`
      directly. The `Z` suffix will be recognized and a UTC value is returned.
    - If the textual value ends with a UTC offset (e.g. `+02:00`), drift first
      uses `DateTime.parse` which respects the modifier but returns a UTC
      datetime. Drift then calls `toLocal()` on this intermediate result to
      return a local value.
    - If the textual value neither has a `Z` suffix nor a UTC offset, drift
      will parse it as if it had a `Z` modifier, returning a UTC datetime.
      The motivation for this is that the `datetime` function in sqlite3 returns
      values in this format and uses UTC by default.

   This behavior works well with the date functions in sqlite3 while also
   preserving "UTC-ness" for stored values.

The mode can be changed with the `store_date_time_values_as_text` [build option]({{ '../Advanced Features/builder_options.md' | pageUrl }}).

Regardless of the option used, drift's builtin support for
[date and time functions]({{ '../Advanced Features/expressions.md#date-and-time' | pageUrl }})
return an equivalent values. Drift internally inserts the `unixepoch`
[modifier](https://sqlite.org/lang_datefunc.html#modifiers) when unix timestamps
are used to make the date functions work. When comparing dates stored as text,
drift will compare their `julianday` values behind the scenes.

#### Migrating between the two modes

While making drift change the date time modes is as simple as changing a build
option, toggling this behavior is not compatible with existing database schemas:

1. Depending on the build option, drift expects strings or integers for datetime
   values. So you need to migrate stored columns to the new format when changing
   the option.
2. If you are using SQL statements defined in `.drift` files, use custom SQL
  at runtime or manually invoke datetime expressions with a direct
  `FunctionCallExpression` instead of using the higher-level date time APIs, you
  may have to adapt those usages.

   For instance, comparison operators like `<` work on unix timestamps, but they
  will compare textual datetime values lexicographically. So depending on the
  mode used, you will have to wrap the value in `unixepoch` or `julianday` to
  make them comparable.

As the second point is specific to usages in your app, this documentation only
describes how to migrate stored columns between the format:

{% assign conversion = "package:drift_docs/snippets/migrations/datetime_conversion.dart.excerpt.json" | readString | json_decode %}

Note that the JSON serialization generated by default is not affected by the
datetime mode chosen. By default, drift will serialize `DateTime` values to a
unix timestamp in milliseconds. You can change this by creating a
`ValueSerializer.defaults(serializeDateTimeValuesAsString: true)` and assigning
it to `driftRuntimeOptions.defaultSerializer`.

##### Migrating from unix timestamps to text

To migrate from using timestamps (the default option) to storing datetimes as
text, follow these steps:

1. Enable the `store_date_time_values_as_text` build option.
2. Add the following method (or an adaption of it suiting your needs) to your
   database class.
3. Increment the `schemaVersion` in your database class.
4. Write a migration step in `onUpgrade` that calls
  `migrateFromUnixTimestampsToText` for this schema version increase.
  __Remember that triggers, views or other custom SQL entries in your database
  will require a custom migration that is not covered by this guide.__

{% include "blocks/snippet" snippets = conversion name = "unix-to-text" %}

##### Migrating from text to unix timestamps

To migrate from datetimes stored as text back to unix timestamps, follow these
steps:

1. Disable the `store_date_time_values_as_text` build option.
2. Add the following method (or an adaption of it suiting your needs) to your
   database class.
3. Increment the `schemaVersion` in your database class.
4. Write a migration step in `onUpgrade` that calls
  `migrateFromTextDateTimesToUnixTimestamps` for this schema version increase.
  __Remember that triggers, views or other custom SQL entries in your database
  will require a custom migration that is not covered by this guide.__

{% include "blocks/snippet" snippets = conversion name = "text-to-unix" %}

Note that this snippet uses the `unixepoch` sqlite3 function, which has been
added in sqlite 3.38. To support older sqlite3 versions, you can use `strftime`
and cast to an integer instead:

{% include "blocks/snippet" snippets = conversion name = "text-to-unix-old" %}

When using a `NativeDatabase` with a recent dependency on the
`sqlite3_flutter_libs` package, you can safely assume that you are on a recent
sqlite3 version with support for `unixepoch`.

## Custom constraints

Some column and table constraints aren't supported through drift's Dart api. This includes `REFERENCES` clauses on columns, which you can set
through `customConstraint`:

```dart
class GroupMemberships extends Table {
  IntColumn get group => integer().customConstraint('NOT NULL REFERENCES groups (id)')();
  IntColumn get user => integer().customConstraint('NOT NULL REFERENCES users (id)')();

  @override
  Set<Column> get primaryKey => {group, user};
}
```

Applying a `customConstraint` will override all other constraints that would be included by default. In
particular, that means that we need to also include the `NOT NULL` constraint again.

You can also add table-wide constraints by overriding the `customConstraints` getter in your table class.

## References

[Foreign key references](https://www.sqlite.org/foreignkeys.html) can be expressed
in Dart tables with the `references()` method when building a column:

```dart
class Todos extends Table {
  // ...
  IntColumn get category => integer().nullable().references(Categories, #id)();
}

@DataClassName("Category")
class Categories extends Table {
  IntColumn get id => integer().autoIncrement()();
  // and more columns...
}
```

The first parameter to `references` points to the table on which a reference should be created.
The second parameter is a [symbol](https://dart.dev/guides/language/language-tour#symbols) of the column to use for the reference.

Optionally, the `onUpdate` and `onDelete` parameters can be used to describe what
should happen when the target row gets updated or deleted.

Be aware that, in sqlite3, foreign key references aren't enabled by default.
They need to be enabled with `PRAGMA foreign_keys = ON`.
A suitable place to issue that pragma with drift is in a [post-migration callback]({{ '../Advanced Features/migrations.md#post-migration-callbacks' | pageUrl }}).

## Views

It is also possible to define [SQL views](https://www.sqlite.org/lang_createview.html)
as Dart classes.
To do so, write an abstract class extending `View`. This example declares a view reading
the amount of todo-items added to a category in the schema from [the example]({{ 'index.md' | pageUrl }}):

{% include "blocks/snippet" snippets = snippets name = 'view' %}

Inside a Dart view, use

- abstract getters to declare tables that you'll read from (e.g. `TodosTable get todos`).
- `Expression` getters to add columns: (e.g. `itemCount => todos.id.count()`).
- the overridden `as` method to define the select statement backing the view.
  The columns referenced in `select` may refer to two kinds of columns:
   - Columns defined on the view itself (like `itemCount` in the example above).
   - Columns defined on referenced tables (like `categories.description` in the example).
     For these references, advanced drift features like [type converters]({{ '../Advanced Features/type_converters.md' | pageUrl }})
     used in the column's definition from the table are also applied to the view's column.

   Both kind of columns will be added to the data class for the view when selected.

Finally, a view needs to be added to a database or accessor by including it in the
`views` parameter:

```dart
@DriftDatabase(tables: [Todos, Categories], views: [CategoryTodoCount])
class MyDatabase extends _$MyDatabase {
```

### Nullability of columns in a view

For a Dart-defined views, expressions defined as an `Expression` getter are
_always_ nullable. This behavior matches `TypedResult.read`, the method used to
read results from a complex select statement with custom columns.

Columns that reference another table's column are nullable if the referenced
column is nullable, or if the selected table does not come from an inner join
(because the whole table could be `null` in that case).

Considering the view from the example above,

- the `itemCount` column is nullable because it is defined as a complex
  `Expression`
- the `description` column, referencing `categories.description`, is non-nullable.
  This is because it references `categories`, the primary table of the view's
  select statement.
