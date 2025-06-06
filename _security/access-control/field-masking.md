---
layout: default
title: Field masking
parent: Access control
nav_order: 95
redirect_from:
 - /security/access-control/field-masking/
 - /security-plugin/access-control/field-masking/
---

# Field masking

If you don't want to remove fields from a document using [field-level security]({{site.url}}{{site.baseurl}}/security/access-control/field-level-security/), you can mask their values. Currently, field masking is only available for string-based fields and replaces the field's value with a cryptographic hash.

Field masking works alongside field-level security on the same per-role, per-index basis. You can allow certain roles to see sensitive fields in plain text and mask them for others. A search result with a masked field might look like the following:

```json
{
  "_index": "movies",
  "_source": {
    "year": 2013,
    "directors": [
      "Ron Howard"
    ],
    "title": "ca998e768dd2e6cdd84c77015feb29975f9f498a472743f159bec6f1f1db109e"
  }
}
```


## Set the salt setting

You can set the salt (a random string used to hash your data) in `opensearch.yml` using the optional `plugins.security.compliance.salt` setting. The salt value must fulfill the following requirements:

- Must be at least 16 characters.
- Use only ASCII characters.

The following example shows a salt value:

```yml
plugins.security.compliance.salt: abcdefghijklmnop
```

Although setting the salt is optional, it is highly recommended.


## Configure field masking

You configure field masking using OpenSearch Dashboards, `roles.yml`, or the REST API.

### OpenSearch Dashboards

1. Choose a role.
1. Choose an index permission.
1. For **Anonymization**, specify one or more fields and press Enter.


### roles.yml

```yml
someonerole:
  index_permissions:
    - index_patterns:
      - 'movies'
      allowed_actions:
        - read
      masked_fields:
        - "title"
        - "genres"
```


### REST API

See [Create role]({{site.url}}{{site.baseurl}}/security/access-control/api/#create-role).


## (Advanced) Use an alternative hash algorithm

By default, the Security plugin uses the BLAKE2b algorithm, but you can use any hashing algorithm that your JVM provides. This list typically includes MD5, SHA-1, SHA-384, and SHA-512.

You can override the default algorithm in `opensearch.yml` using the optional default masking algorithm setting `plugins.security.masked_fields.algorithm.default`, as shown in the following example:

```yml
plugins.security.masked_fields.algorithm.default: SHA-256
```
OpenSearch 3.x contains a bug fix to apply the default BLAKE2b algorithm correctly. You can override the default algorithm in OpenSearch 3.x to continue to produce the same masked values as OpenSearch 1.x and 2.x  in `opensearch.yml` using the optional default masking algorithm setting `plugins.security.masked_fields.algorithm.default`, as shown in the following example:

```yml
plugins.security.masked_fields.algorithm.default: BLAKE2B_LEGACY_DEFAULT
```

To specify a different algorithm, add it after the masked field in `roles.yml`, as shown in the following:

```yml
someonerole:
  index_permissions:
    - index_patterns:
      - 'movies'
      allowed_actions:
        - read
      masked_fields:
        - "title::SHA-512"
        - "genres"
```


## (Advanced) Pattern-based field masking

Rather than creating a hash, you can use one or more regular expressions and replacement strings to mask a field. The syntax is `<field>::/<regular-expression>/::<replacement-string>`. If you use multiple regular expressions, the results are passed from left to right, like piping in a shell, as shown in the following example:

```yml
hr_employee:
  index_permissions:
    - index_patterns:
      - 'humanresources'
      allowed_actions:
        - read
      masked_fields:
        - 'lastname::/.*/::*'
        - '*ip_source::/[0-9]{1,3}$/::XXX::/^[0-9]{1,3}/::***'
someonerole:
  index_permissions:
    - index_patterns:
      - 'movies'
      allowed_actions:
        - read
      masked_fields:
        - "title::/./::*"
        - "genres::/^[a-zA-Z]{1,3}/::XXX::/[a-zA-Z]{1,3}$/::YYY"

```

The `title` statement changes each character in the field to `*`, so you can still discern the length of the masked string. The `genres` statement changes the first three characters of the string to `XXX` and the last three characters to `YYY`.


## Effect on audit logging

The read history feature lets you track read access to sensitive fields in your documents. For example, you might track access to the email field of your customer records. Access to masked fields are excluded from read history, because the user only saw the hash value, not the clear text value of the field.
