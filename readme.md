# eve - REST API for Humans - demo

[eve home page](http://python-eve.org/index.html) 

PHILOSOPHY
---
You have data => Need a full featured REST API => you got it
powered by Flask, mongoDB, redis

GET IT STARTED
----
1. create virtualenv
2. 

FEATURES
---
######MONGO FILTERS: 
`?where ={"lastname": "Doe"}`
######PYTHON FILTERS: 
`?where=lastname=="Doe"`
######SORTING:
sorty by 'TOTAL', descending order: `?sort=[("total": -1)]`
######PAGINATION:
`?max_results=20&page=2`
######PROJECTIONS:
return all fields but 'AVATAR': `?projection={"avatar": 0}`
only return 'LASTNAME': `?projection={"lastname": 1}`
######EMBEDDED RESOURCES:
`?embedded={"author": 1}`
```sh
$ curl -i <url>
HTTP/1.1 200 OK
{
    "title": "Book Title",
    "description": "book description",
    "author": <uuid>
}
# request embedded author
$ curl -i <url>?embedded={"author": 1}
HTTP/1.1 200 OK
{
    "title": "Book Title",
    "description": "book description",
    "author": {
        "firstname": "Mark",
        "lastname": "Green",
    }
}
```
######BUILT-IN FOR ALL RESPONSES
`application/json`
`application/xml`
######HATEOAS
```sh
{
    "_links": {
        "self": {
            "href": "/people",
            "title": "people"},
        "parent": {
            "href": "/",
            "title": "home"},
        "next": {
            "href": "/people?page=2",
            "title": "next page"},
        "last": {
            "href": "/people?page=10",
            "title": "last page"},
    }
}```
also custom render classes
JSON-LD/HAL/SIREN*


######DOCUMENT VERSION
```
?version=3
?version=all
?version=diffs
```
also EVE_DOCS: generates documentation for EVE APIS in HTML and JSON 
######FILE STORAGE
```sh
# in settings.py
accounts = {
    'name': {'type': 'string'},
    'pic': {'type': 'media'},
    ...
}
######RATE LIMITING
Response example
```sh
$ curl -i <url>

HTTP/1.1 200 OK
X-RateLimit-Limit: 1
X-RateLimit_remaining: 0
X-RateLimit-Reset: 1390486659

$ curl -i <url>
HTTP/1.1 429 TOO MANY REQUESTS
```
######IF-MODIFIED-SINCE
Allow clients to only request Non-Cached Content
If-Modified-Since: Wed, 05 Dec 2014 09:53:07 GMT
"Please return modified data since <date> or 304"
######BULK INSERTS
inserts multiple documents with single quote
```sh
$ curl -d '
[
    {"firstname": "a", "lastname": "c"},
    {"firstname": "b", "lastname": "d"},
]
'
-H 'Content-Type: application/json' <url>

# bulk inserts response
# coherence mode off: only meta fields are returned otherwise returns all fields 
[
    {
        "_status": "OK",
        "_updated": "Thu, 22 Nov 2014 15:22:27 GMT",
        "_id": <uuid>,
        "_links": {"self": {..}}
    },
    ...
]
```

######DATA INTEGRITY CONCURRENCY CONTROL
no overwriting documents with obsolete versions
```sh
# IF-MATCH MISSING
$ curl -X PATCH -i <url> -d {"firstname": "roland"}
HTTP/1.1 403 FORBIDDEN

# ETAG MISSING
$ curl -X PATCH -i <url>
    -H "If-Match: <obsolete_etag>"
    -d '{"firstname": "roland"}'
HTTP/1.1 412 PRECONDITION FAILED

# ETAG MATCH
$ curl -X pATH -i <url>
    -H "If-Match: <valid_etag>",
    -d '{"firstname": "roland"}'
HTTP/1.1 200 OK
```
######DATA VALIDATION & CUSTOM VALIDATION
1. add custom data types
2. add custome validtion logic
######Custom EVENT HOOK
1. POST on_insert/in_inserted
2. GET on_fetch/on_fetched
3. ...


######GeoJSON
Support and validation for GeoJSON types

######CUSTOM DATA LAYERS
SQL Alchemy, Elastic Search, Mongo

######POST / GET file
also supportes
```
$ curl F "name=testfile" -F "pic=@profile.jpb" <url>
HTTP/1.1 200 OK

$curl -i <url>
HTTP/1.1 200 OK
{
    "name": "john",
    "pic": "/9j/4QAYRXhpZgAASUkqAAgAAA....." # return as base64
}
# respoinse with EXTEBDED_MEDIA_INFO: TRUE
{
    "name": "john",
    "pic": {
        "file": "/9j/4QAYRXhpZgAASUkqAAgAAA.....", # return as base64
        "content_type": "image/jpeg",
        "name": "profile.jpg",
        "length": 8129
}
```

run.py:
```python
from eve import Eve
app = Eve()

if __name__ == '__main__':
    app.run()
```
settings.py:
```python
# just a couple API endpoints with no custom
# schema or rules. Will just dump from people and book db collections
DOMAIN = {
    'people': {}
    'books': {}
}

# connect to a mongo instance
MONGO_HOST = 'localhost'
MONGO_PORT = 27017
MONGO_USERNAME = 'user'
MONGO_PASSWORD = 'user'
MONGO_DBNAME = 'apitest'

# add some validation rules
DOMAIN['poeple']['schema'] = {
    'name': {
        'type': 'string',
        'maxlength': 50,
        'unique': True}
    'email': {
        'type': 'string',
        'regex': '^\S+@\S+$'},
    'location': {
        'type': 'dict',
        'schema': {
            'address': {'type': 'string'},
            'city': {'type': 'string'}}},
    'born': {'type': 'datetime'}
}

# allow write access to API endpoints
# /people
# 'POST': add/create one more more items
RESROUCE_METHOD = ['GET', 'POST']
# /people/<id>
# 'PATCH': edit item
# 'PUT': replace item
ITEM_METHODS = ['GET', 'PATCH', 'PUT', 'DELETE']

# more config options
DOMAIN['people'].update(
    {
        'item_title': 'person',
        'cache_control': 'max-page=10, must-revalidate',
        'cache_expires': 10,
        'additional_lookup': {
            'url': 'regex)"[\w]+")',
            'field': 'name'
        }
    }
)

# Rate Limiting
# Rate limit on GET requests: 1 requests 1 minutes window (per client)
RATE_LIMIT_GET = (1, 60)
```
terminal: run server
```sh
$ python run.py
* Running on http://127.0.0.7:5000/
```
terminal: make request
```sh
# "_links": HATEOAS at work
# "parent": clients can explore the API programatically
# "title": client can utilize info
$ curl -i http://127.0.0.7:5000/people
{
    "_items": [],
    "_links": {
        "self": {
            "href": "127.0.0.7:5000/people",
            "title": "people"
        },
        "parent": {
            "href": "127.0.0.1:5000",
            "title": "home"
        }
    }
}
```

### Todo's

 - Write Tests
 - Rethink Github Save
 - Add Code Comments
 - Add Night Mode
