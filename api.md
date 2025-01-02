<h1 style="text-align: center">API Design</h1>

The purpose of this document is to explain the design of the apis under each microservice in stirzz app. All of these apis should be designed to achieve some use-case defined in the [Use Cases](./use-case.md) document while complying with the [Data Model Design](./data-model.md) and [High Level Design](./hld.md).

## Inventory Service
Inventory service includes apis which help manage inventory operations, i.e. crud inventory items and sending change updates to relevant services. Below is the list of these apis with their design details.

### Create or Update Inventory Item
This api should be used to create or update an inventory item.

**path**
: `PUT: /inventory/{:sku}`

**request-body for create**
```
{
    "inStock": "$integer"
}
```

**request-body for increment**
```
{
    "inStock": "$integer",
    "add": "$integer",
    "etag": "$string"
}
```

**request-body for decrement**
```
{
    "inStock": "$integer",
    "remove": "$integer",
    "etag": "$string"
}
```
- *(upon uncountable items in inventory, the data model should be updated to accomodate that)*

#### Implementation Steps:
1. validate request
    1. [assert idempotancy](#http-put-request-idempotancy-assertion)
    1. validate body
        1. **if:** is create request
            1. assert no field other than `inStock` is present
        1. **else:** it's update request
            1. assert field `inStock` is present
            1. assert at least one from `add` or `remove` fields is present
            1. assert `add` and `remove` fields not present together at the same time i.e. (they are mutually exclusive)
1. **if:** is create request
    1. create new inventory item and store in database
    1. respond with **201** general response
1. **else:** it's update request
    1. **if:** `add` field is present in request
        1. add the value from `add` field in request to current inventory item's `inStock` field
    1. **else:** `remove` field is present in request
        1. **if:** value of `remove` is greater than current inventory item's `stock`
            1. respond with **422** general response
        1. **else:** remove from the current inventory item's `inStock` field the value of `remove` field present in request
    1. save the change to database
    1. respond with **204** general response

- _(current inventory prefetched in step 1.1 should be reused in update steps)_
- _(responses should comply with [general responses](#general-responses))_

## HTTP GET Response

### Locale Selection
By default server should use `accept-language` header to select response language, if query parameter `lang` is present then it should override the `accept-language` header.

### ETags to support idempotent methods
To help idempotent methods the HTTP GET method should return etags for the resources contained in the response. Now for single resource the ETag should be present in both the `ETag` response header as well as in the response payload itself, for multiple resource response the ETags should be present only in response payload and the `ETag` header should not be present in response at all. The etag value generation should be deffered to ORM using an integer version column in database _(see [Data Model](./data-model.md))_, while the final etag value should be with a prepended 'v' character i.e. if data model returned version to be '1' the etag field in response should be 'v1'.

## HTTP PUT Request Idempotancy Assertion
For asserting idempotency in HTTP PUT requests use `ETag` and `If-Match` headers. Clients will have `ETag` value from earlier server responses (see [HTTP GET Response](#http-get-response)), Now if clients needs to send an update single resource request, they should send `If-Match` header with it's value equal to the `ETag` from the latest server response for this resource, and if both values match strongly (character by character) then the request is eligible for update otherwise should be responded with precondition failure.

1. **if:** `If-Match` header is present
    1. get resource `ETag` (etag value for current version of resource)
    1. **if:** `ETag` does not match strongly (character by character) with `If-Match` header value
        1. respond with 412
    1. **else:** proceed with update
1. **else-if:** resource already exist
    1. respond with 409
1. **else:** proceed with create

## General Responses
### 201 Created
when new resource created successfully

#### Response
**status code**
: 201

**headers**\
If single resource was created then following headers should be present

- `ETag`: indicating the version of newly created entity
- `Location`: containing the relative path of the newly created entity

### 204 No Content
when existing resource updated successfully

#### Response
**status code**
: 204

**headers**\
If single resource was updated then following headers should be present

- `ETag`: indicating the version of newly created entity
- `Location`: containing the relative path of the newly created entity

### 412 Precondition Failed
when precondition failed i.e. current resource `ETag` did not match with `If-Match` header value

#### Response
**status code**
: 412

**response body**
: response body should be [error response](#error-response) with no mention of current ETag at all, the client should be forced to get complete entity in a subsequent request

### 409 Conflict
when resource already exists when trying to create new resource 

#### Response
**status code**
: 409

**response body**
: response body should be [error response](#error-response) mentioning what the conflict is

### 422 Unprocessable Content
when request payload is not valid

#### Response
**status code**
: 422

**response body**
: response body should be [error response](#error-response) containing information about what is wrong in the request payload

## Error Response
The error responses should contain following information.

**~~type~~**
:

**title**
: static text indicating type of error

**~~status~~**
:

**detail**
: text describing specific details of error

**~~instance~~**
:

_(~~strikethrough~~ fields may be available after completing design for [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) compliance)_

**example response:**
```
{
    "title": "Precondition Failed",
    "detail": "Cannot update resource as resouce does not exist"
}
```