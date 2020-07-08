# Scoutr

A simple way to put an API in front of a DynamoDB backend.

## Sample implementation

An sample implementation of this project is provided in the [example](example) folder.

## Requirements

At minimum, two Dynamo tables are required for this to work: an auth table and a groups table. Additionally, an optional
audit log table can be used to track all API calls and changes to records in the data table. The configuration of
each table is detailed next.

### Auth Table
The auth table must have a partition key of `id`. The table name does not matter, as this is passed in during
instantiation.

### Groups Table
The groups table must have a partition key of `group_id`. The table name does not matter, as this is passed in during
instantiation.

### Audit Log Table
The audit log table must have a partition key of `time`. It should also have a TTL attribute of `expire_time`
configured. The table name does not matter, as this is passed in during instantiation. If a value is not specified, it
is assumed that no audit logs should be kept.

## Access Control

Scoutr provides full access control over the endpoints a set of users is permitted to call and the output that is
returned. This is done using field filters, field exclusions, and permitted endpoints, which are outlined in the next
section.

This access control functionality is implemented at both a user and a group level. A user can be a member of zero or
more groups. The implementation of [auth identifiers](#auth-identifier) and [groups](#groups) is outlined in their
respective sections.

The two types of access control supported are via API Gateway or via OIDC. Helper functions have been created for
each access control type to assist with passing the correct request format into Scoutr.

### API Gateway Authentication
For API Gateway authentication, the request format is generated by the `build_api_gateway_request` method.

#### Example
Refer to the [example serverless endpoint](example/api_gateway/example/endpoints/list.py)

### OIDC Authentication
It is assumed that there is an Apache server running in front of the application that performs OIDC authentication
and passes the OIDC claims as headers.

The simplest method to setup the API is to use [Flask API](https://www.flaskapi.org/). Helper methods have been
provided to make the setup as simple as possible. The [`init_flask`](scoutr/flask/routes.py#L11) method
automatically generates the belows endpoints:
- GET `/user/` - Returns information about the authenticated user
- POST `/user/has-permission/` - Determine if user has permission to access an endpoint. The body of this request should
    contain `method` and `path` keys as JSON.
- GET `/<primary_list_endpoint>/` - Primary endpoint used to list data. The value of `primary_list_endpoint` is determined
    by an argument passed to `init_flask()`
- GET `/audit/` - List and search all audit logs
- GET `/audit/<item>/` - List audit logs for a particular resource
- GET `/history/<item>/` - Show history for a particular resource
- POST `/search/<search_key>/` - Search endpoint that allows searching by any key for one or more values. The body of
    this request should be a JSON list of values.

#### Example
Refer to the [example flask application](example/flask/main.py)

### Concepts

#### Field filters

List of field filters to apply to queries by this group. Each item in this list must be structured as:

If the type of `value` is a string, it will be filtered using a `field = value` operation. To support multiple
values for a single field, tf the type of `value` is a list, it will be filtered using a
`field IN ['value1', 'value2', ..., 'valueN']` operation. When multiple field filters are specified, they are
combined together using an `AND` operation.

##### Syntax
```json
[
    {"field": "field1", "value": "filter_value"},
    {"field": "field2", "value": ["value1", "value2"]},
]
```

#### Field exclusions

Field exclusions allow for excluding one or more fields from the output of all queries. These fields are from any output
during the post-processing phase of all queries. Additionally, if a user attempts to create or update an item that
contains a field from this list, the operation will be denied.

##### Syntax
```json
[
    "field1",
    "field2"
]
```

#### Permitted endpoints

Before taking any action, every call from API gateway is validated to ensure the user has permissions to
perform the call. For convenience, regular expressions can be used within the `endpoint` field.

##### Syntax
```json
[
    {"method": "GET|POST|PUT|DELETE", "endpoint": "/endpoint"},
    {"method": "GET|POST|PUT|DELETE", "endpoint": "^/endpoint2/.+$"}
]
```

### Groups

A group object be made up of:
- `group_id` - Identifier for the group
- `permitted_endpoints` - Optional list of permitted endpoints
- `filter_fields` - Optional list of field filters
- `exclude_fields` - Optional list of field exclusions
- `update_fields_permitted` - Optional list of the only fields that can be updated
- `update_fields_restricted` - Optional list of fields to restrict updates for

The name of the group table must be passed in to the constructor.

#### Example
```json
{
    "group_id": "read-only",
    "permitted_endpoints": [
        {
            "endpoint": "^/account/.+$",
            "method": "GET"
        },
        {
            "endpoint": "^/accounts.*$",
            "method": "GET"
        },
        {
            "endpoint": "^/search/.+$",
            "method": "POST"
        }
    ],
    "exclude_fields": [
        "field1"
    ],
    "update_fields_permitted": [
        "field4"
    ],
    "update_fields_restricted": [
        "field5"
    ],
    "filter_fields": [
        {
            "field": "field2",
            "value": "value1"
        },
        {
            "field": "field3",
            "value": [
                "value2",
                "value3"
            ]
        }
    ]
}
```

### Auth Identifier

#### Types

There are three types of accepted authentication identifiers:
- USERNAME
- OIDC_GROUP
- API_KEY

Though not required, it is recommended for each object type to have a `type` key that corresponds to its
authentication type (OIDC_GROUP, USERNAME, or API_KEY).

The field requirements for each object type are outlined in the following sections

##### USERNAME
- id (partition key) - this is the user's username

Though not required, it is recommended to also include a `name` field containing the user's full name to make it
easier to identify the user at a glance.

##### OIDC_GROUP
- id (partition key) - this is expected to be the group id

Though not required, it is recommended to also include a `name` field containing the group's display name to make it
easier to identify the group at a glance.

If a user is a member of more than one OIDC group, the permissions granted by each configured group will be combined
together to generate the effective permissions applied to the user.

##### API_KEY
- id (partition key) - this is the api key id
- name
- username
- email

#### Groups

Optionally, each auth object can include a `groups` object, which should be a list of group ids that the user is a
member of:
```
{
    "groups": [
        "read-only",
        "view-audit-logs"
    ]
}
```

Any permissions defined in the groups are combined together to make up the user's permissions. In addition, the same
permissions that a group defines (`filter_fields`, `exclude_fields`, `update_fields_permitted`,
`update_fields_restricted`, `permitted_endpoints`) can be expressed at the user level. These permissions will be
combined together with the permissions outlined in the groups the user is a member of. Permissions defined at the user
level **DO NOT** override those specified at the group level - they are combined.

If a user is a member of multiple OIDC groups, the permissions are combined such that
- Any `filter_fields` definied in a child `groups` block are combined together with an `AND` expression
- All of the combined permissions in each OIDC group the user is a member of are combined together with an `OR` expression

The name of the user table must be passed in to the constructor.

### Audit Logs

For every authorized, successful call to the API, an entry will be logged in the audit log table. Each record will
follow the below format:

```json
{
  "action": "CREATE|UPDATE|DELETE|GET|LIST|SEARCH|{CUSTOM-ACTION}",
  "body": {
    "key": "value"
  },
  "method": "HTTP method from API gateway",
  "path": "/endpoint/path",
  "path_params": {
    "key": "value"
  },
  "query_params": {
    "key": "value"
  },
  "resource": {
    "key": "value"
  },
  "time": "2019-10-04T18:44:30.166635",
  "user": {
    "api_key_id": "ID",
    "name": "John Doe",
    "source_ip": "1.2.3.4",
    "username": "222222222",
    "user_agent": "curl"
  }
}
```

The following fields may not be included or may not have values for all types of actions:
- body
- query_params
- path_params
- resource

## Endpoint Structure

The helper methods within Scoutr assume that your API consists of the following endpoint types:
- [List all records](#list)
- [List all unique values for a key](#list-by-unique-key)
- [Search multiple values for a single search key](#search)
- [Get single item by key](#get)
- [Update single item by key](#update)
- [Delete single item by key](#delete)
- [List all audit logs)(#list-audit-logs)
- [View item history](#history)

### List

The list all items endpoint will return a list of all items within the backend that the user has permission to see
and that meet any specified filter criteria.

### List by Unique Key

The list by unique key endpoint is provided as a means to display all unique values for a single search key. It is
implemented by specifying a value for the `unique_key` argument of the `list_table()` method. The simplest way to
implement this without duplicating code is to use a `UniqueKey` environment variable that defaults to a value of `None`
when the environment variable is not specified. Then, just configure your "list by unique key" Lambda with that
environment variable.

#### Serverless Example
```yml
# Unique listing of all values of the `status` key that the user is permitted to see
list-statuses:
  handler: endpoints.list.main
  events:
    - http:
        path: statuses
        method: get
        private: true
  environment:
    UniqueKey: status
```

#### Implementation Example
```python
def lambda_handler(event, context):
    path_params = event.get('pathParameters', {}) or {}
    query_params = event.get('queryStringParameters', {}) or {}
    api = DynamoAPI(
        table_name=os.getenv('TableName'),
        auth_table_name=os.getenv('AuthTable'),
        group_table_name=os.getenv('GroupTable')
    )
    data = api.list_table(
        request=build_api_gateway_request(event),
        unique_key=os.getenv('UniqueKey'),
        path_params=path_params,
        query_params=query_params
    )
```

### Search

Lookup information about multiple items (POST `/search/{search_key}`)
```
[
    "123456789012"
]
```

### Get

Retrieve a single record from the backend. The `get_item()` method accepts two arguments:
- `key` - the key to search by
- `value` - the value to search by

If this returns more than one record, it will throw a `BadRequestException`. If no records are
returned, a `NotFoundException` will be thrown.

### Create

The `create()` method accepts an `item` argument, with `item` being the `dict` of the data to
be inserted. It also accepts a `field_validation` argument in order to perform validation on
all the supplied data. Refer to the [data validation](#data-validation) section for more
information.

### Update

The `update()` method accepts a couple of arguments:

**`partition_key`**
Mapping of the partition key to value. For instance, if the table's partition key is `id`, it is expected this mapping
would be:

```python
{'id': 'value'}
```

**`data`**
Dictionary of fields to be updated

**`field_validation`**
Dictionary of fields to perform validation against. Refer to the [data validation](#data-validation) section for more
information.

**`condition`**
Conditional expression to apply on updates. This should be an instance of boto3's
[`ConditionExpression`](https://www.programcreek.com/python/example/103724/boto3.dynamodb.conditions.Attr). If the
condition expression does not pass, a `BadRequestException` will be thrown.

**`condition_failure_message`**

By default, if the condition expression does not pass, it will return an error to the user stating
"Conditional check failed". However, if this parameter is supplied, it will be returned to the user instead.

### Delete

The `delete()` method accepts a couple of arguments:

**`partition_key`**
Mapping of the partition key to value. For instance, if the table's partition key is `id`, it is expected this mapping
would be:

```python
{'id': 'value'}
```

**`condition`**
Conditional expression to apply on deletions. This should be an instance of boto3's
[`ConditionExpression`](https://www.programcreek.com/python/example/103724/boto3.dynamodb.conditions.Attr). If the
condition expression does not pass, a `BadRequestException` will be thrown.

**`condition_failure_message`**

By default, if the condition expression does not pass, it will return an error to the user stating
"Conditional check failed". However, if this parameter is supplied, it will be returned to the user instead.

### List all audit logs

The `list_audit_logs()` method accepts:

**`search_params`**
Any search parameters to apply

**`query_params`**
Query parameters from API Gateway

### History

**`key`**
Resource key to search on

**`value`**
Resource value to search on

**`query_params`**
Query parameters from API Gateway

**`actions`**
Actions to filter on

## Filtering

There are two levels of filtering that are supported:
- Path-based filtering
- Querystring-based filtering

The `list_table()` method accepts both `path_params` and `query_params` as arguments. These are intended to
contain the values of `pathParameters` and `queryStringParameters`, respectively, that API Gateway passed into Lambda.

### Dynamic path filters

The `list_table()` method also supports dynamic path filtering. When `search_key` and `search_value` are passed into
the method as `path_params`, it will dynamically modify the path parameters to construct a search filter where

```
search_key = search_value
```

To configure this in API Gateway, setup path parameters on the resource:
```
/endpoint/{search_key}/{search_value}
```

Or when using serverless:

```yml
events:
  - http:
      path: endpoint
      method: get
      private: true
  - http:
      path: endpoint/{search_key}/{search_value}
      method: get
      private: true
```

When using the dynamic path filters, there is no need to construct additional endpoints that support filtering by a
specific key. However, using this method provides no limitations over what fields can be used as a filter. If that is a
concern for your API, you will need to construct static path filters.

### Static path filters

Static path filters can be constructed in a similar manner to the dynamic path filters, except that the search key is
manually specified:

```
/endpoint/status/{status}
```

In order to properly work, the path variable must _exactly_ match the key in the backend table that you want to perform
the filter against.

### Querystring Filters

In addition to path filters, querystring filtering is also supported. The `list_table()` endpoint accepts all
querystrings via the `query_params` argument. Each querystring should be a `field_name=search_value` format:

```
/endpoint?status=Active&field3=value2
```

Path parameters **always** take precedence over querystring parameters. The below query:

```
/endpoint/field2/value1?status=Active&field2=value2
```

Would result in this filter criteria:

```
field2 = value1 AND status = Active
```

#### Magic Operators

For more complex queries, querystring search supports the below magic operations:
- `in` (value is in list)
- `ne` (not equal)
- `startswith` (string starts with)
- `contains` (string contains)
- `notcontains` (string does not contain)
- `exists` (attribute exists)
- `gt` (greater than)
- `lt` (less than)
- `gte` (greater than or equal)
- `lte` (less than or equal)
- `between` (value is between)

To use a magic operator, append `__operator` to the key name. For example:

To search for all items with the `field1` key containing the phrase "val"

```
/items?field1__contains=val
```

To search for all items with the `field1` key starting with the phrase "va"

```
/items?field1__startswith=va
```

Usage of all the magic operators is straightforward, with the exception of the `in` and `betweeen` operators. The `in`
operators checks to see if the the value is included in a list of options. It should follow the JSON list syntax:

```
/items?field1__in=["value1", "value2"]
```

The `between` operator checks to see if the value is, inclusively, between a low and high value. It should also follow
a JSON list syntax:

```
/items?num__between=[0, 3]
```

It also works for string values, such as two dates:

```
/items?date__between=["2019-01-01", "2019-12-31"]
```

To find items that have an attribute:

```
/items?name__exists=true
```

To search for items that do not have an attribute:

```
/items?name__exists=false
```

## Data validation

For convenience, support for data validation on all create and update calls is supported. In order to implement the
validation, a dictionary should be passed to the `field_validation` argument of the `create()` or `update()` methods
of `DynamoAPI`. The syntax of this dictionary is outlined below.

On `create()` calls, all items specified in the `field_validation` dictionary are assumed to be required fields. If a
field is missing from the user input, an error will be thrown saying that the field is required.

### Syntax
```python
FIELD_VALIDATION = {
    'field_name_1': lambda value, item, existing_item: callable_that_returns_a_bool,
    'field_name_2': lambda value, item, existing_item: callable_that_returns_a_dict,
    'field_name_3': callable_that_returns_a_bool,
    'field_name_4': callable_that_returns_a_dict
}
```

The key of each item in the dictionary should match a field name that you want to perform validation against. The
corresponding value for the key should be a callable that either returns a bool or a dict formatted as:
```
{
    'result': boolean that indicates whether this field was valid or not,
    'message': 'custom error message to return to the user'
}
```

The callable that you provide can either be a function or a lambda. The method signature of both options **must accept**
three arguments:
- `value` - Contains the input value for this field
- `item` - Contains the entire data object that was passed from the user
- `existing_item` - Contains the existing data object in Dynamo. This will only have a value on update calls. For
    create calls, this will be `None`.

### Example
```python
def validate_user(value, item, existing_item=None):
    if isinstance(existing_item, dict):
        item_type = existing_item.get('type')
    else:
        item_type = item.get('type')

    if not item_type:
        return {
            'result': False,
            'message': 'Type field is required'
        }

    if item_type == '1':
        return {
            'result': re.match('^\d{9}$', value),
            'message': 'Invalid user for type %s' % item_type
        }
    elif item_type == '2':
        return {
            'result': re.match('^.+@example.com$', value),
            'message': 'Invalid user for type %s' % item_type
        }
    else:
        return False

FIELD_VALIDATION = {
    'user': validate_user,
    'type': lambda value, item, existing_item: DynamoAPI.value_in_list(
        value=value,
        valid_options=['1', '2'],
        option_name='type'
    ),
    'description': lambda value, item, existing_item: isinstance(value, str),
    'name': lambda value, item, existing_item: {
        'result': re.match('^\w+ \w+$', value),
        'message': 'Invalid name format'
    }
}
```

## Sentry support

Support for sentry is built-in to Scoutr. Breadcrumbs are automatically added in at key points in the execution.
