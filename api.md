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
- *(upon uncountable items e.g. liquids in inventory, the data model should be updated to accomodate that i.e. datatype should change e.g. float than integer)*

#### Implementation Steps:
1. validate request
    1. [assert Idempotency](#http-put-request-idempotency-assertion)
    1. validate body
        1. **if: (is create request)**
            1. assert no fields other than `req.inStock` is present
        1. **else:** # is update request
            1. assert field `req.inStock` is present and has same value as current stock value in database
            1. assert at least one from `req.add` or `req.remove` fields is present
            1. assert `req.add` and `req.remove` fields not present together at the same time i.e. (they are mutually exclusive)
1. **if: (is create request)**
    1. create new inventory item and store in database
    1. respond with **201** general response
1. **else:** # is update request
    1. **if: (`req.add` field present)**
        1. add the value from `req.add` to current inventory item's `inStock` field
    1. **else:** # `req.remove` field present
        1. **if: (`req.remove` > current `stock`)** # value of `req.remove` is greater than current inventory item's `stock`
            1. respond with **422** general response
        1. **else:** # `req.remove` <= current `stock` 
            1. remove from the current inventory item's `inStock` field the value of `req.remove` field present in request
    1. save the change to database
    1. respond with **204** general response

- _(create vs update can be determined while processing idempotency assertion in step 1.1)_
- _(current inventory prefetched in step 1.1 should be reused in update steps)_
- _(responses should comply with [general responses](#general-responses) and [Error Response](#error-response))_

## Idempotency
HTTP methods like GET, PUT, DELETE are idempotent methods by nature, which means if a request is sent to the server first time using any of these methods and whatever change on application resources occurs on server if any, then this change should not occur ever again if this same request was repeated second, third or any number of times. For example if a request to make stock increase for a prodcut is made to make it 100 from 10 i.e. _10 + 90 = 100_ then if this HTTP PUT request is sent 100 times the stock count should remain 100 rather than _10 + (100 x 90) = 9,010_.

## HTTP GET Response

### Locale Selection
By default server should use `accept-language` header to select response language, if query parameter `lang` is present then it should override the `accept-language` header.

### ETags to support idempotent methods
To help idempotent methods the HTTP GET method should return etags for the resources contained in the response. Now for single resource the ETag should be present in both the `ETag` response header as well as in the response payload itself, for multiple resource response the ETags should be present only in response payload and the `ETag` header should not be present in response at all. The etag value generation should be deffered to ORM using an integer version column in database _(see [Data Model](./data-model.md))_, while the final etag value should be with a prepended 'v' character i.e. if data model returned version to be '1' the etag field in response should be 'v1'.

## HTTP PUT Request Idempotency Assertion
HTTP PUT requests are used to either create a single resource or update a single resource. As HTTP PUT is idempotent in nature we need to make sure calling HTTP PUT multiple time results exactly the same on resources as calling it one time. For asserting idempotency in HTTP PUT requests use `ETag` and `If-Match` headers. Clients will have `ETag` value from earlier server responses (see [HTTP GET Response](#http-get-response)), Now if clients needs to send an update single resource request, they should send `If-Match` header with it's value equal to the `ETag` from the latest server response for this resource, and if both values match strongly (character by character) then the request is eligible for update otherwise should be responded with precondition failure.

1. **if: (`If-Match` header present)** # implies update
    1. get resource `ETag` (etag value for current version of resource)
    1. **if: (`ETag` mismatch)** # `ETag` does not match strongly (character by character) with `If-Match` header value
        1. respond with 412
    1. **else: (`ETag` matches)** proceed with update
1. **else-if: (resource already exist)** # `If-Match` header is absent and create resourse already exists
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