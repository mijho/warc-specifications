---
title: CDX for non-GET requests
type: specification
status: wild
version: draft
---

# CDX for non-GET requests

Originally CDX files were only used to index web archives containing  GET requests. As browser based capture methods
can record non-GET requests such as those generated by JavaScript a way for CDX records to differentiate based on
request method and request body is needed. This specification describes the mechanism used for encoding the request 
method and body in the CDX key by appending additional query parameters to the URL.

> **Compatibility Note**
>
> This specification aims to be compatible with pywb 2.6.7 running on Python 3.7 or later. 
> Prior to Python 3.7 the urlencode() function percent encoded the character "~". Prior to Python 3.6 dictionaries did
> not preserve insertion order which may cause query parameters to be re-ordered.

## Overview

[TODO: To be written]

## Encoding the request method

If the request method is not `GET` it must be appended as the value of query parameter `__wb_method`.

If the URL does not have a query string a `?` must be added:

    http://example.org/         => http://example.org/?__wb_method=POST

If the URL already has a query string the `__wb_method` parameter must be added at the end after a `&` separator:

    http://example.org/?page=1  => http://example.org/?page=1&__wb_method=POST

Even if the query string already ends in `&` another separator must still be added:

    http://example.org/?foo&    => http://example.org/?foo&&__wb_method=POST

## Encoding the request body

Encoding the request body depends on the content-type.

| Content-Type                      | Primary Encoding | Fallback Encoding |
|-----------------------------------|------------------|-------------------|
| application/json                  | JSON             |                   |
| application/x-amf                 | AMF              |                   |
| application/x-www-form-urlencoded | form urlencoded  | binary            |
| multipart/*                       | form multipart   | binary            |
| text/plain                        | JSON             | binary            |
| *                                 | binary           |                   |

### AMF encoding

[TODO: To be written]

### Binary encoding

The request body is encoded as Base64 ([RFC 4648](https://tools.ietf.org/html/rfc4648)) and appended to the query string as the `__wb_post_data` parameter.

> **Example**
> 
> Original request:
> 
>     POST /chat HTTP/1.0
>     Host: example.org
>     Content-Length: 5
>
>     hello
>
> Encoded URL:
>
>     http://example.org/chat?__wb_method=POST&__wb_post_data=aGVsbG8=

### Form urlencoded encoding

If the body is valid UTF-8 it must be URL decoded, URL encoded [TODO: specify] and then appended to the output.
If UTF-8 decoding fails the binary encoding method must be used instead.

[TODO: example]

### Form multipart encoding

The body must be decoded as form data per [RFC 2388](https://datatracker.ietf.org/doc/html/rfc2388) and then
URL encoded [TODO: specify]. If the body is not a valid multipart/form-data message then the binary encoding method
must be used instead.

[TODO: example]

### JSON encoding

The request must be parsed as JSON ([RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259)) and then apply
the following algorithm with an empty string as the initial value of *name*.

To **encode a JSON *value***, given a *name* and an initially-empty map *nameCounts* of strings to integers:

1. If *value* is a JSON object:
   1. Recursively encode each member of the object passing member's name as *name* and the member's value as *value*.
2. If *value* is a JSON array:
   1. Recursively encode each element of the array passing the current value of *name* as 
      *name* and the value of the element as *value*.
3. Otherwise:
   1. Define the string *encodedValue* as:
      1. If *value* is JSON true then the string "True".
      2. If *value* is JSON false then the string "False".
      3. If *value* is JSON null then the string "None".
      4. If *value* is a JSON string then the urlencoding [TODO: specify] of the string.
      5. If *value* is a JSON number then the number encoded as a decimal string.~~~~
   2. If *nameCounts* contains the integer *count* for *name*:
      1. Increment *count* by 1.
      2. Store *count* as the new count for *name* in *nameCounts*.
      3. Append the string "&*name*.*count*_=*encodedValue*" to the output.
   3. Otherwise, if *nameCounts* does not contain *name*:
      1. Store the integer 1 in *nameCounts* for *name*.
      2. Append the string "&*name*=*encodedValue*" to the output.

> **Example**
> 
> Original request:
> 
>     POST /events HTTP/1.0
>     Host: example.org
>     Content-Type: application/json
> 
>     {
>        "type": "event",
>        "id": 44.0,
>        "values": [true, false, null],
>        "source": {
>           "type": "component",
>           "id": "a+b&c= d",
>           "values": [3, 4]
>        }
>     }
> 
> Encoded URL:
> 
>     http://example.org/events?__wb_method=POST&type=event&id=44.0&values=True
>     &values.2_=False&values.3_=None&type.2_=component&id.2_=a%2Bb%26c%3D+d
>     &values.4_=3&values.5_=4
