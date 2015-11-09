# Coverage Data REST API Core Specification

## 1. Introduction

This API specification is structured in such a way that a server implementation
can evolve naturally, beginning with no custom implementation at all.
It is based on largely independent concepts of which some or all can be integrated,
depending on which level of sophistication is necessary, which in turn is
determined by the amount of data to be served, the type of users, and their access patterns.

It is important to realize that there is no requirement to implement one or the other
aspect detailed below. This is made possible by self-describing and therefore
advertising the offered capabilities that the server offers to the client in each and every resource
as metadata, without relying on a fixed structure like an API root document.

A specific API that is implemented in a server and follows aspects of this specification
may be loosely named "Coverage Data REST API", however, since such API is merely
a collection of supported concepts, it is more important to refer and uniquely identify
those concepts. The reason for that is extensibility and interoperability.
This specification is meant as a starting place which solves common issues, but a
natural extension with other external concepts is explicitly allowed and recommended.

### Notes

In the following, the term "Coverage Data" stands both for a single Coverage,
but also for a collection of Coverages.
A Coverage can contain multiple measured/computed parameters, like wind speed
and wind direction.

As an example, a netCDF file (extension `.nc`) can contain a single or multiple
coverages. The same is true for a CoverageJSON file (extension `.covjson`).
A GeoTIFF file (extension `.geotiff`) contains a single coverage.

In example responses, HTTP headers that are not relevant may be omitted.

## 2. Static resources

In its simplest form, Coverage Data is made available as one or more resources,
served with the correct media type. The resources can be static files.

Example serving a netCDF file:
```sh
$ curl http://example.com/coveragedata.nc

HTTP/1.1 200 OK
Content-Type: application/x-netcdf

[binary netcdf]
```

Example serving a GeoTIFF file:
```sh
$ curl http://example.com/coveragedata.geotiff

HTTP/1.1 200 OK
Content-Type: image/tiff

[binary geotiff]
```

## 3. Content negotiation

If the same Coverage Data should be made available in different formats (encodings),
then it is advisable to use [content negotiation](https://en.wikipedia.org/wiki/Content_negotiation)
for that purpose.
This requires more support on the server side but makes sure that the
Coverage Data as a concept stays at a single URL (which is important
in the linked web).

Example:
```sh
$ curl http://example.com/coveragedata -H "Accept: application/x-netcdf"

HTTP/1.1 200 OK
Content-Type: application/x-netcdf

[binary netcdf]
```
```sh
$ curl http://example.com/coveragedata -H "Accept: image/tiff"

HTTP/1.1 200 OK
Content-Type: image/tiff

[binary geotiff]
```

The idea is that the client is aware of the formats it supports and therefore
knows the media types to include in the "Accept" header.
This is how web browsers work as well, they know HTML, so they
specifically request that media type.

### A note on copy-pasting URLs and expectations

A slight inconvenience with this approach is that it doesn't easily allow
a human to copy-paste URLs into a browser address bar and "download" the data,
since he cannot select the format directly.
A common way out is to also make the different formats available under URLs with file extensions
(see the "Static resources" section), which then would always deliver the corresponding format.

However, this case gets more complex once the API grows and, for example,
serves just the first 100 coverages in a big collection, relying on a client that
understands how to "page" through the collection.
Then the expectation of simply "downloading" all the data in the correct format
by appending a file extension might lead to confusing results ("Where's the rest?").
In that sense, file extensions are likely not always the answer, and the specific
use case of downloading data dumps may need to be modeled differently.

## 4. Collection elements as separate resources

Some formats allow that the coverages inside a collection are split off into separate resources
and referenced from the collection.

If possible, this separation should be exploited, instead of just serving a single file
with all coverages fully embedded. The advantage is that having separate resources for each
coverage allows to link to them, and also allows clients to load only the coverages they
are interested in (since the collection resource might only include overview data then).
Note that this does not mean that there can't be a resource which
offers the collection with full coverage data embedded (more on that in later sections).

Example serving a CoverageJSON collection as separate resources:
```sh
$ curl http://example.com/coveragecollection -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json

{
  "@context": "http://coveragejson.org",
  "id": "http://example.com/coveragecollection",
  "type": "CoverageCollection",
  "parameters": {...},
  "coverages": [{
    "id": "http://example.com/coveragecollection/coverage1",
    "type": "GridCoverage",
    "title": "First coverage",
    "domain": "http://example.com/coverage1_domain",
    "ranges": {"TEMP": "http://example.com/coveragecollection/coverage1/TEMP"} 
  }, {
    "id": "http://example.com/coveragecollection/coverage2",
    "type": "GridCoverage",
    "title": "Second coverage",
    "domain": "http://example.com/coveragecollection/coverage2/domain",
    "ranges": {"SPEED": "http://example.com/coveragecollection/coverage2/SPEED"}
  }]
}
```
```sh
$ curl http://example.com/coveragecollection/coverage1 -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://example.com/coveragecollection>; rel="collection"

{
  "@context": "http://coveragejson.org",
  "type": "GridCoverage",
  "id": "http://example.com/coveragecollection/coverage1",
  "title": "First coverage",
  "domain": {...},
  "ranges": {"TEMP": {...}}
}
```

It is recommended to include the above `Link` header in coverage resources of
the collection to provide context and a way of navigation independent of what might be
stored inside the coverage format itself.

Whether to include the full coverage data in a collection resource or just
provide links to it must be decided case-by-case.
It is advisable that the data volume is kept to an amount that web clients
can still handle easily, which typically means up to a few megabytes at most.

## 5. Paged collection resources

Coverage data can become big quite quickly.
Therefore, different strategies are needed for clients to handle such data in a
convenient way. One such strategy is to offer paged collection resources.

Example serving a paged CoverageJSON collection:
```sh
$ curl http://example.com/coveragecollection -H "Accept: application/prs.coverage+json"

HTTP/1.1 303 See Other
Location: http://example.com/coveragecollection?page=1
Content-length: 0
```
```sh
$ curl http://example.com/coveragecollection?page=1 -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://example.com/coveragecollection>; rel="canonical"
Link: <http://example.com/coveragecollection?page=1>; rel="first"
Link: <http://example.com/coveragecollection?page=2>; rel="next"
Link: <http://example.com/coveragecollection?page=221>; rel="last"

{
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org"
  ],
  "id": "http://example.com/coveragecollection",
  "type": "CoverageCollection",
  "coverages": [...],
  "view" : {
    "id" : "#pagination",
    "@graph" : {
      "id" : "http://example.com/coveragecollection?page=1",
      "type" : "PartialCollectionView",
      "totalItems" : 22021,
      "itemsPerPage" : 100,
      "first" : "http://example.com/coveragecollection?page=1",
      "next" : "http://example.com/coveragecollection?page=2",
      "last" : "http://example.com/coveragecollection?page=221"
    }
  }
}
```
```sh
$ curl http://example.com/coveragecollection?page=2 -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://example.com/coveragecollection>; rel="canonical"
Link: <http://example.com/coveragecollection?page=1>; rel="first"
Link: <http://example.com/coveragecollection?page=1>; rel="prev"
Link: <http://example.com/coveragecollection?page=3>; rel="next"
Link: <http://example.com/coveragecollection?page=221>; rel="last"

{...}
```

There are several things to consider here:
- If there would be just a single page, then none of the above would happen.
  There would be no redirect to page one and there would be no navigational metadata.
  Paging only comes into effect when there is more than one page.
- Navigational metadata has to be included in the response of each page.
  If the format itself allows to include those (as above inside the `"view"` property)
  in an interoperable way then this should be done.
  In addition, matching Link headers may be included as well.
  However, if the format does not allow to include navigational metadata, then
  the Link headers are required.
- A server is not required to support all navigation patterns. This means that
  "first" and "last" are optional. A server can also just support forward traversal.
- If JSON-LD is used as a format, then the navigational metadata should be included
  as above using the Hydra ontology in a non-default graph. The default graph should
  not be used to logically separate the actual data from the control metadata.
- If the format supports resource identifiers (as above), then the collection elements
  have to be associated to the collection resource and *not* the page resource.
  Only the navigational metadata may be associated with the page resource.
  If the format does not support resource identifiers, then a Link header with "canonical"
  is required that links back to the collection resource, otherwise it is optional but
  recommended.

## 6. Embedded resources

In coverage data resources, certain data may not be included by default and instead
be linked. For example, a collection resource may not include the actual domain and range
data of the coverages, but instead just metadata and appropriate URLs to fetch the data.
This is similar to a shopping website, where product overviews are given in search results
that then contain links to the full descriptions.

In some cases, it is useful to include the full data within a given resource to prevent
additional server requests and the transfer of partially duplicate data.
A typical example is to request data of a big number of coverages that
by themselves are fairly small in data volume. Instead of fetching a small collection resource
followed by fetching hundreds of small coverage resources, one could fetch a single bigger collection
resource that includes all necessary data.

A server may offer support for such client preferences but it does not have to.
The recommended way to offer such functionality is described below.

Example:
```sh
$ curl http://example.com/coveragecollection

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://coveragejson.org/def#Domain>; rel="http://coverageapi.org/ns#CanInclude"
Link: <http://coveragejson.org/def#Range>; rel="http://coverageapi.org/ns#CanInclude"
Vary: Prefer

{... domain and range are not embedded by default ...}
```
```sh
$ curl http://example.com/coveragecollection -H "Prefer: include=\"http://coveragejson.org/def#Domain http://coveragejson.org/def#Range\"" \
                                             -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://coveragejson.org/def#Domain>; rel="http://coverageapi.org/ns#CanInclude"
Link: <http://coveragejson.org/def#Range>; rel="http://coverageapi.org/ns#CanInclude"
Vary: Prefer

{... domain and range are embedded now ...}
```
This is based on [RFC7240](https://tools.ietf.org/html/rfc7240) and
the [include](http://www.w3.org/TR/ldp/#prefer-parameters) parameter defined within [LDP](http://www.w3.org/TR/ldp/).
A standard way to advertise available preferences to the client does not exist yet.
In the example above, a custom predicate `http://coverageapi.org/ns#CanInclude` in a Link header is used for that purpose.

Note that the above method requires a server implementation of 
[CORS "preflight"](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing#Preflight_example) requests
which browsers will send when using the `Prefer` header for cross-domain requests.

If the above method using the `Prefer` header is not suitable for a specific API implementation,
then an alternative based on URL query parameters may be used.

Example:
```sh
$ curl http://example.com/coveragecollection -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json

{
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org",
    "api": "http://coverageapi.org/ns#api"
  ],
  "id": "http://example.com/coveragecollection",
  "type": "CoverageCollection",
  "coverages": [...],
  "api": {
    "id" : "#api",
    "@graph" : {
      "type": "IriTemplate",
      "template": "http://example.com/coveragecollection{?include}",
      "mappings": [{
        "type": "IriTemplateMapping",
        "variable": "include",
        "property": {
          "id": "http://coverageapi.org/ns#include",
          "comment": "A comma separated string of one or both: domain, range. Determines what is included in the resource.",
          "range": "xsd:string"
        },
        "required": false
      }]
    }
  }
}
```
```sh
$ curl http://example.com/coveragecollection?include=domain,range -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://example.com/coveragecollection>; rel="canonical"

{...}
```

Metadata on how to build the URL for embedding data has to be included in each resource
that supports it, as above inside the `"api"` property.
If JSON-LD is used as a format, then that metadata should be included
as above using the Hydra ontology in a non-default graph. The default graph should
not be used to logically separate the actual data from the control metadata.
