# search

The `search` command retrieves documents from the index. The `search` command can only be used as the first command in the PPL query.

## Syntax

```
search source=[<remote-cluster>:]<index> [search-expression]
```

- **search**: search keyword, which could be ignored
- **index**: mandatory. Search command must specify which index to query from. The index name can be prefixed by `<cluster name>:` for cross-cluster search
- **search-expression**: optional. Search expression that gets converted to OpenSearch [query_string](https://docs.opensearch.org/latest/query-dsl/full-text/query-string/) function which uses [Lucene Query Syntax](https://lucene.apache.org/core/2_9_4/queryparsersyntax.html)

## Cross-Cluster Search

Cross-cluster search lets any node in a cluster execute search requests against other clusters. Refer to [Cross-Cluster Search](admin/cross_cluster_search.rst) for configuration.

## Example 1: Text Search

### Basic Text Search

Search for a single unquoted term:

```ppl
search ERROR source=otellogs
| sort @timestamp
| fields severityText, body
| head 1
```

Expected output:

```text
fetched rows / total rows = 1/1
+--------------+---------------------------------------------------------+
| severityText | body                                                    |
|--------------+---------------------------------------------------------|
| ERROR        | Payment failed: Insufficient funds for user@example.com |
+--------------+---------------------------------------------------------+
```

### Phrase Search

Requires quotes for multi-word exact match:

```ppl
search "Payment failed" source=otellogs
| fields body
```

Expected output:

```text
fetched rows / total rows = 1/1
+---------------------------------------------------------+
| body                                                    |
|---------------------------------------------------------|
| Payment failed: Insufficient funds for user@example.com |
+---------------------------------------------------------+
```

### Implicit AND with Multiple Terms

Unquoted literals are combined with AND:

```ppl
search user email source=otellogs
| sort @timestamp
| fields body
| head 1
```

Expected output:

```text
fetched rows / total rows = 1/1
+--------------------------------------------------------------------------------------------------------------------+
| body                                                                                                               |
|--------------------------------------------------------------------------------------------------------------------|
| Executing SQL: SELECT * FROM users WHERE email LIKE '%@gmail.com' AND status != 'deleted' ORDER BY created_at DESC |
+--------------------------------------------------------------------------------------------------------------------+
```

> **Note:** `search user email` is equivalent to `search user AND email`. Multiple unquoted terms are automatically combined with AND.

### Special Characters in Search Terms

Enclose in double quotes for terms which contain special characters:

```ppl
search "john.doe+newsletter@company.com" source=otellogs
| fields body
```

Expected output:

```text
fetched rows / total rows = 1/1
+--------------------------------------------------------------------------------------------------------------------+
| body                                                                                                               |
|--------------------------------------------------------------------------------------------------------------------|
| Email notification sent to john.doe+newsletter@company.com with subject: 'Welcome! Your order #12345 is confirmed' |
+--------------------------------------------------------------------------------------------------------------------+
```

### Mixed Phrase and Boolean

Combine phrase search with boolean operators:

```ppl
search "User authentication" OR OAuth2 source=otellogs
| sort @timestamp
| fields body
| head 1
```

Expected output:

```text
fetched rows / total rows = 1/1
+----------------------------------------------------------------------------------------------------------+
| body                                                                                                     |
|----------------------------------------------------------------------------------------------------------|
| [2024-01-15 10:30:09] production.INFO: User authentication successful for admin@company.org using OAuth2 |
+----------------------------------------------------------------------------------------------------------+
```

## Example 2: Boolean Logic and Operator Precedence

### Boolean Operators

Use OR to match multiple conditions:

```ppl
search severityText="ERROR" OR severityText="FATAL" source=otellogs
| sort @timestamp
| fields severityText
| head 3
```

Expected output:

```text
fetched rows / total rows = 3/3
+--------------+
| severityText |
|--------------|
| ERROR        |
| FATAL        |
| ERROR        |
+--------------+
```

Use AND to combine conditions:

```ppl
search severityText="INFO" AND `resource.attributes.service.name`="cart-service" source=otellogs
| fields body
| head 1
```

Expected output:

```text
fetched rows / total rows = 1/1
+----------------------------------------------------------------------------------+
| body                                                                             |
|----------------------------------------------------------------------------------|
| User e1ce63e6-8501-11f0-930d-c2fcbdc05f14 adding 4 of product HQTGWGPNH4 to cart |
+----------------------------------------------------------------------------------+
```

### Operator Precedence

Operator precedence (highest to lowest): Parentheses → NOT → OR → AND

```ppl
search severityText="ERROR" OR severityText="WARN" AND severityNumber>15 source=otellogs
| sort @timestamp
| fields severityText, severityNumber
| head 2
```

Expected output:

```text
fetched rows / total rows = 2/2
+--------------+----------------+
| severityText | severityNumber |
|--------------+----------------|
| ERROR        | 17             |
| ERROR        | 17             |
+--------------+----------------+
```

> **Note:** The above evaluates as `(severityText="ERROR" OR severityText="WARN") AND severityNumber>15`

## Example 3: NOT vs != Semantics

### != Operator

The `!=` operator requires the field to exist and not equal the value:

```ppl
search employer!="Quility" source=accounts
```

Expected output:

```text
fetched rows / total rows = 2/2
+----------------+-----------+--------------------+---------+--------+--------+----------+-------+-----+-----------------------+----------+
| account_number | firstname | address            | balance | gender | city   | employer | state | age | email                 | lastname |
|----------------+-----------+--------------------+---------+--------+--------+----------+-------+-----+-----------------------+----------|
| 1              | Amber     | 880 Holmes Lane    | 39225   | M      | Brogan | Pyrami   | IL    | 32  | amberduke@pyrami.com  | Duke     |
| 6              | Hattie    | 671 Bristol Street | 5686    | M      | Dante  | Netagy   | TN    | 36  | hattiebond@netagy.com | Bond     |
+----------------+-----------+--------------------+---------+--------+--------+----------+-------+-----+-----------------------+----------+
```

### NOT Operator

The `NOT` operator excludes matching conditions and includes null fields:

```ppl
search NOT employer="Quility" source=accounts
```

Expected output:

```text
fetched rows / total rows = 3/3
+----------------+-----------+----------------------+---------+--------+--------+----------+-------+-----+-----------------------+----------+
| account_number | firstname | address              | balance | gender | city   | employer | state | age | email                 | lastname |
|----------------+-----------+----------------------+---------+--------+--------+----------+-------+-----+-----------------------+----------|
| 1              | Amber     | 880 Holmes Lane      | 39225   | M      | Brogan | Pyrami   | IL    | 32  | amberduke@pyrami.com  | Duke     |
| 6              | Hattie    | 671 Bristol Street   | 5686    | M      | Dante  | Netagy   | TN    | 36  | hattiebond@netagy.com | Bond     |
| 18             | Dale      | 467 Hutchinson Court | 4180    | M      | Orick  | null     | MD    | 33  | daleadams@boink.com   | Adams    |
+----------------+-----------+----------------------+---------+--------+--------+----------+-------+-----+-----------------------+----------+
```

> **Note:** Notice that the `NOT` operator includes the row where `employer` is null (Dale Adams), while `!=` excludes it.
