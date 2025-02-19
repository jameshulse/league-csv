---
layout: default
title: Converting a CSV into a JSON payload
---

# JSON conversion

<p class="message-notice">Since version <code>9.10.0</code>.</p>

The `JsonEncoder` converts a collection of serializable structures into a JSON string payload.
While PHP provides the `JsonSerializable` interface and the package `TabularDataReader` classes
implement it, if the document to serialize is "too" big it will make your script run out of
memory because the CSV needs to be handle in full in memory.

To resolve this issue, the package introduces a lightweight JSON encoder for collection.
This converter is built to handle huge files in a memory efficient way via a minimalist public API.

<p class="message-warning">The encoder works with the <code>iterable</code> structure only. If
your collection also supports the <code>JsonSerializable</code> interface, its method will be
ignored.</p>

## Settings

Prior to converting the records into a JSON payload, you may wish to configure how the encoding
will preserve records information. As with the other converters included in the package,
the `JsonEncoder` is immutable. Everytime you change a settings a new instance
will be return without changing the current instance.

### Json encoding options

```php
public JsonEncoder::flags(int $flags): self
```

The method supports all the flags from `json_encode`. To make sure an exception is always thrown instead
of substituting some unencodable values in some specific cases, the following flags are applied, by default:

```php
JSON_THROW_ON_ERROR & ~JSON_PARTIAL_OUTPUT_ON_ERROR
```

The `JSON_PARTIAL_OUTPUT_ON_ERROR` flag **MUST** be explicitly set if you want its feature to be used
during conversion. This behaviour deviates from the default behaviour from `json_encode`.

Please refer to the [JSON constants page](https://www.php.net/manual/en/json.constants.php) on the PHP
website for more information.

You can access the current setting using the converter public readonly property `flags`.

```php
use League\Csv\JsonEncoder;

$converter = JsonEncoder::create();
$converter->flags; // returns JSON_THROW_ON_ERROR & ~JSON_PARTIAL_OUTPUT_ON_ERROR;
```

### Json conversion depth

```php
public JsonEncoder::depth(int $depth): self
```

Just like, `json_encode`, it is possible to define the conversion depth variable. The method will
throw an exception if the value is lesser than `1`.

You can access the current setting using the converter public readonly property `deopth`.

```php
use League\Csv\JsonEncoder;

$converter = JsonEncoder::create();
$converter->depth; // returns 512;
```

### Collection offset process

```php
public JsonEncoder::includeOffset(): self
public JsonEncoder::excludeOffset(): self
```

These methods tell the converter to include or exclude the iterable offset when converting
the document. If the offset needs to be present in the returned JSON payload, the returned JSON
payload will be an object whose members key is the collection offset and the member value the json
serialized member value.

<p class="message-info">The default is to exclude the offset information.</p>

You can access the current setting using the converter public readonly property `preserveOffset`.

```php
use League\Csv\JsonEncoder;

$converter = JsonEncoder::create();
$converter->includeOffset; // returns false;
```

## Conversion

```php
public JSONConverter::encode(iterable $records): Generator<string>;
public JSONConverter::encodeToFile(iterable $records, SplFileObject $file): int;
public JSONConverter::encodeToStream(iterable $records, resource $stream): int;
public JSONConverter::encodeToPath(iterable $records, string $path, string $open_mode = 'w', resource $context = null): int;
```

All the methods accept an `iterable` object whose members **MUST** support being encoded
using PHP's `json_encode` function and returns either directly a `Generator` with the
JSON payload being generated or saved in the designed persistence layer progressively.

```php
use League\Csv\JsonEncoder;
use League\Csv\Statement;
use League\Csv\Reader;

$csv = Reader::createFromPath('path/to/files/prenoms.csv')
    ->setDelimiter(';')
    ->setHeaderOffset(0);

//We are using the CharsetConverter to avoid the JsonEncoder throwing an exception
//because the CSV file contains non-compatible utf8 characters and thus will fail
//json encoding if the charset converter is not used.

CharsetConverter::addTo($csv, 'iso-8859-15', 'utf-8');

$stmt = Statement::create()
    ->where(fn (array $record): bool => str_starts_with($record['prenoms'], 'Anaë'))
    ->orderBy(fn (array $recordA, array $recordB): int => $recordB['nombre'] <=> $recordA['nombre'])
    ->offset(0)
    ->limit(3)
;

$bytes = JsonEncoder::create()
    ->includeOffset()
    ->flags(JSON_PRETTY_PRINT | JSON_NUMERIC_CHECK)
    ->encodeToStream($stmt->process($csv), STDOUT);

echo PHP_EOL, $bytes, ' bytes were written.', PHP_EOL;

// will output in your console the following
// {
//     "1": {
//         "prenoms": "Ana\u00eblle",
//         "nombre": 38,
//         "sexe": "F",
//         "annee": 2004
//     },
//     "2": {
//         "prenoms": "Ana\u00eblle",
//         "nombre": 31,
//         "sexe": "F",
//         "annee": 2005
//     },
//     "8": {
//         "prenoms": "Ana\u00eblle",
//         "nombre": 30,
//         "sexe": "F",
//         "annee": 2008
//     }
// }
// 356 bytes were written.

$bytes = JsonEncoder::create()->encodeToPath($csv, '/path/storage/file.json');
echo PHP_EOL, $bytes, ' bytes were written.', PHP_EOL;
// will output
// 612875 bytes were written.
// and save in your file the full JSON payload in a single line progressively
// [{"prenoms":"Aaron","nombre":"55","sexe":"M","annee":"2004"},...,{"prenoms":"Zohra","nombre":"6","sexe":"F","annee":"2012"}]
```
