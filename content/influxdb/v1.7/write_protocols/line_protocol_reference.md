---
title: InfluxDB line protocol reference
aliases:
    - /influxdb/v1.7/write_protocols/write_syntax/
menu:
  influxdb_1_7:
    name: InfluxDB line protocol reference
    weight: 10
    parent: Write protocols
---


The InfluxDB line protocol is a text based format for writing points to InfluxDB.

## InfluxDB line protocol

### Syntax

```
<measurement>[,<tag_key>=<tag_value>[,<tag_key>=<tag_value>]] <field_key>=<field_value>[,<field_key>=<field_value>] [<timestamp>]
```

Lines separated by the newline character `\n` represent a single point
in InfluxDB. Line protocol is whitespace-sensitive.

>**Note** Line protocol does not support the newline character `\n` within tag values or field values.

### Description of syntax

InfluxDB line protocol informs InfluxDB of the data's measurement, tag set, field set,
and timestamp.

| Element | Optional/Required | Description | Type<br>(See [data types](#data-types) for more information.) |
| :-------| :---------------- |:----------- |:----------------
| [Measurement](/influxdb/v1.7/concepts/glossary/#measurement) | Required | The measurement name. InfluxDB accepts one measurement per point. | String
| [Tag set](/influxdb/v1.7/concepts/glossary/#tag-set) | Optional | All tag key-value pairs for the point.  | [Tag keys](/influxdb/v1.7/concepts/glossary/#tag-key) and [tag values](/influxdb/v1.7/concepts/glossary/#tag-value) are both strings.
| [Field set](/influxdb/v1.7/concepts/glossary/#field-set) | Required. Points must have at least one field. | All field key-value pairs for the point. | [Field keys](/influxdb/v1.7/concepts/glossary/#field-key) are strings. [Field values](/influxdb/v1.7/concepts/glossary/#field-value) can be floats, integers, strings, or Booleans.
| [Timestamp](/influxdb/v1.7/concepts/glossary/#timestamp) | Optional. InfluxDB uses the server's local nanosecond timestamp in UTC if the timestamp is not included with the point. | The timestamp for the data point. InfluxDB accepts one timestamp per point. | Unix nanosecond timestamp. Specify alternative precisions with the [InfluxDB API](/influxdb/v1.7/tools/api/#write-http-endpoint).

> #### Performance and Setup Tips:
>
* Sort tags by key before sending them to the database.
The sort should match the results from the
[Go bytes.Compare function](http://golang.org/pkg/bytes/#Compare).
* Use the coarsest
[precision](/influxdb/v1.7/tools/api/#write-http-endpoint) possible for timestamps.
This can result in significant improvements in compression.
* Use the Network Time Protocol (NTP) to synchronize time between hosts.
InfluxDB uses a host's local time in UTC to assign timestamps to data; if
hosts' clocks aren't synchronized with NTP, the timestamps on the data written
to InfluxDB can be inaccurate.

## Data types

| Datatype     | Element(s)  | Description  |
| :----------- | :------------------------ |:------------ |
| Float | Field values |  IEEE-754 64-bit floating-point numbers. This is the default numerical type. Examples: `1`, `1.0`, `1.e+78`, `1.E+78`. |
| Integer | Field values | Signed 64-bit integers (-9223372036854775808 to 9223372036854775807). Specify an integer with a trailing `i` on the number. Example: `1i`. |
| String | Measurements, tag keys, tag values, field keys, field values | Length limit 64KB. |
| Boolean | Field values | Stores TRUE or FALSE values.<br><br>TRUE write syntax:`[t, T, true, True, TRUE]`.<br><br>FALSE write syntax:`[f, F, false, False, FALSE]` |
| Timestamp | Timestamps | Unix nanosecond timestamp. Specify alternative precisions with the [InfluxDB API](/influxdb/v1.7/tools/api/#write-http-endpoint). The minimum valid timestamp is `-9223372036854775806` or `1677-09-21T00:12:43.145224194Z`. The maximum valid timestamp is `9223372036854775806` or `2262-04-11T23:47:16.854775806Z`. |

#### Boolean syntax for writes and queries

Acceptable Boolean syntax differs for data writes and data queries.
For more information, see
[Frequently asked questions](/influxdb/v1.7/troubleshooting/frequently-asked-questions/#why-can-t-i-query-boolean-field-values).

#### Field type discrepancies

Within a measurement, a field's type cannot differ within a
[shard](/influxdb/v1.7/concepts/glossary/#shard), but it can differ across
shards.
For how field value type discrepancies can affect `SELECT *` queries, see
[How does InfluxDB handle field type discrepancies across shards?](/influxdb/v1.7/troubleshooting/frequently-asked-questions/#how-does-influxdb-handle-field-type-discrepancies-across-shards).

### Examples

#### Write the field value `1.0` as a float to InfluxDB

```sql
> INSERT mymeas value=1.0
```

#### Write the field value `1` as a float to InfluxDB

```sql
> INSERT mymeas value=1
```

#### Write the field value `-1.234456e+78` as a float to InfluxDB

```sql
> INSERT mymeas value=-1.234456e+78
```

InfluxDB supports field values specified in scientific notation.

#### Write the field value `1` as an integer to InfluxDB

```sql
> INSERT mymeas value=1i
```

#### Write the field value `stringing along` as a string to InfluxDB

```sql
> INSERT mymeas value="stringing along"
```

Always double quote string field values. More on quoting [below](#quoting).

#### Write the field value `true` as a Boolean to InfluxDB

```sql
> INSERT mymeas value=true
```

Do not quote Boolean field values.
The following statement writes `true` as a string field value to InfluxDB:

```sql
> INSERT mymeas value="true"
```

#### Attempt to write a string to a field that previously accepted floats

If the timestamps on the float and string are stored in the same shard:

```sql
> INSERT mymeas value=3 1465934559000000000
> INSERT mymeas value="stringing along" 1465934559000000001
ERR: {"error":"field type conflict: input field \"value\" on measurement \"mymeas\" is type string, already exists as type float"}
```

If the timestamps on the float and string are not stored in the same shard:

```sql
> INSERT mymeas value=3 1465934559000000000
> INSERT mymeas value="stringing along" 1466625759000000000
>
```

## Quoting, special characters, and additional naming guidelines

### Quoting

| Element | Double quotes | Single quotes |
| :------ | :------------ |:------------- |
| Timestamp | Never | Never |
| Measurements, tag keys, tag values, field keys | Never* | Never* |
| Field values | Double quote string field values. Do not double quote floats, integers, or Booleans. | Never |

\* InfluxDB line protocol allows users to double and single quote measurement names, tag
keys, tag values, and field keys.
It will, however, assume that the double or single quotes are part of the name,
key, or value.
This can complicate query syntax (see the example below).

#### Examples

##### Invalid line protocol - Double quote the timestamp

```sql
> INSERT mymeas value=9 "1466625759000000000"
ERR: {"error":"unable to parse 'mymeas value=9 \"1466625759000000000\"': bad timestamp"}
```

Double quoting (or single quoting) the timestamp yields a `bad timestamp`
error.

##### Semantic error - Double quote a Boolean field value

```sql
> INSERT mymeas value="true"
> SHOW FIELD KEYS FROM "mymeas"
name: mymeas
------------
fieldKey	 fieldType
value		   string
```

InfluxDB assumes that all double quoted field values are strings.

##### Semantic error - Double quote a measurement name

```sql
> INSERT "mymeas" value=200
> SHOW MEASUREMENTS
name: measurements
------------------
name
"mymeas"
> SELECT * FROM mymeas
> SELECT * FROM "mymeas"
> SELECT * FROM "\"mymeas\""
name: "mymeas"
--------------
time				                        value
2016-06-14T20:36:21.836131014Z	 200
```

If you double quote a measurement in line protocol, any queries on that
measurement require both double quotes and escaped (`\`) double quotes in the
`FROM` clause.

### Special characters

You must use a backslash character `\` to escape the following special characters:

* In string field values, you must escape:
  * double quotes  
  * backslash character

For example, `\"` escapes double quote.
   
>#### Note on backslashes:
>
* If you use multiple backslashes, they must be escaped. Influx interprets backslashes as follows:
  *	`\` or `\\` interpreted as `\` 
  *	`\\\` or `\\\\` interpreted as `\\` 
  * `\\\\\` or `\\\\\\` interpreted as `\\\`, and so on

* In tag keys, tag values, and field keys, you must escape: 
  * commas
  * equal signs
  * spaces

For example, `\,` escapes a comma.
    
* In measurements, you must escape:
  * commas  
  * spaces

You do not need to escape other special characters.

#### Examples

##### Write a point with special characters

```sql
> INSERT "measurement\ with\ quo⚡️es\ and\ emoji",tag\ key\ with\ sp🚀ces=tag\,value\,with"commas" field_k\ey="string field value, only \" need be esc🍭ped"
```

The system writes a point where the measurement is `"measurement with quo⚡️es and emoji"`, the tag key is `tag key with sp🚀ces`, the
tag value is `tag,value,with"commas"`, the field key is `field_k\ey` and the field value is `string field value, only " need be esc🍭ped`.

### Additional naming guidelines

`#` at the beginning of the line is a valid comment character for line protocol.
InfluxDB will ignore all subsequent characters until the next newline `\n`.

Measurement names, tag keys, tag values, field keys, and field values are
case sensitive.

InfluxDB line protocol accepts
[InfluxQL keywords](/influxdb/v1.7/query_language/spec/#keywords)
as [identifier](/influxdb/v1.7/concepts/glossary/#identifier) names.
In general, we recommend avoiding using InfluxQL keywords in your schema as
it can cause
[confusion](/influxdb/v1.7/troubleshooting/errors/#error-parsing-query-found-expected-identifier-at-line-char) when querying the data.

The keyword `time` is a special case.
`time` can be a
[continuous query](/influxdb/v1.7/concepts/glossary/#continuous-query-cq) name,
database name,
[measurement](/influxdb/v1.7/concepts/glossary/#measurement) name,
[retention policy](/influxdb/v1.7/concepts/glossary/#retention-policy-rp) name,
[subscription](/influxdb/v1.7/concepts/glossary/#subscription) name, and
[user](/influxdb/v1.7/concepts/glossary/#user) name.
In those cases, `time` does not require double quotes in queries.
`time` cannot be a [field key](/influxdb/v1.7/concepts/glossary/#field-key) or
[tag key](/influxdb/v1.7/concepts/glossary/#tag-key);
InfluxDB rejects writes with `time` as a field key or tag key and returns an error.
See [Frequently Asked Questions](/influxdb/v1.7/troubleshooting/frequently-asked-questions/#time) for more information.

## InfluxDB line protocol in Practice

See the [Tools](/influxdb/v1.7/tools/) section for how to
write line protocol to the database.

### Duplicate points

A point is uniquely identified by the measurement name, tag set, and timestamp.
If you submit line protocol with the same measurement, tag set, and timestamp,
but with a different field set, the field set becomes the union of the old
field set and the new field set, where any conflicts favor the new field set.

For a complete example of this behavior and how to avoid it, see
[How does InfluxDB handle duplicate points?](/influxdb/v1.7/troubleshooting/frequently-asked-questions/#how-does-influxdb-handle-duplicate-points)

