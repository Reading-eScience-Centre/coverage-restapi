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
which may be static files served by a simple server.

**Requirement:** The correct media type must be returned for the formats that are offered.

**Example serving a netCDF file:**
```sh
$ curl http://example.com/coveragedata.nc

HTTP/1.1 200 OK
Content-Type: application/x-netcdf

[binary netcdf]
```

**Example serving a GeoTIFF file:**
```sh
$ curl http://example.com/coveragedata.geotiff

HTTP/1.1 200 OK
Content-Type: image/tiff

[binary geotiff]
```

## 3. Content negotiation

**Recommendation:** If the same Coverage Data should be made available in different formats,
then [content negotiation](https://en.wikipedia.org/wiki/Content_negotiation) should be used.

Content negotiation requires more support on the server side but makes sure that the
Coverage Data as a concept stays at a *single* URL (which is important
in the linked web).

**Example using content negotiation:**
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
specifically request that format.

**Pro Tip:** Offer HTML as format as well so that *humans* can explore the coverage data as well.

### A note on copy-pasting URLs

A slight inconvenience with this approach is that it doesn't easily allow
a human to copy-paste URLs into a browser address bar and "download" the data in a given format,
since he cannot select the format directly.
A common way out is to also make the different formats available under URLs with file extensions
(see the "[Static resources](#2-static-resources)" section),
which then would always deliver the corresponding format and side-step content negotiation.
This approach is also common with HTML pages when doing content negotiation based on languages
using the `Accept-Language` header. The initial page would typically redirect to a page that
has the language encoded within the URL, making it possible to link to that specific language
version. As long as humans and machines are aware of the distinction between the canonical
resource and a specialized version of it there should be no harm in following such patterns.

TODO possibly make this more concrete, suggest rel=canonical Links

## 4. Collection elements as separate resources

Some coverage data formats allow to group coverages in collections,
for example the netCDF-CF and the CoverageJSON formats, but not the GeoTIFF format.
Of those formats, some allow to split off the individual coverages as separate resources
and reference them from the collection, possibly including some limited metadata about
each referenced coverage within the collection itself.

**Recommendation:** If the coverage data format supports collections and allows to split off
individual coverages into separate resources, then this separation should be exploited,
instead of just serving a single file with all coverages fully embedded.

The advantage is that having separate resources for each
coverage allows to link to them, and also allows clients to load only the coverages they
are interested in (since the collection resource might only include overview data then).
Note that this does not mean that there can't be a resource which
offers the collection with full coverage data embedded (more on that in later sections).

**Example serving a CoverageJSON collection as separate resources:**
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
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org",
    { "inCollection": { "@reverse": "member" } }
  ],
  "type": "GridCoverage",
  "id": "http://example.com/coveragecollection/coverage1",
  "title": "First coverage",
  "domain": {...},
  "ranges": {"TEMP": {...}},
  "inCollection": {
    "id": "http://example.com/coveragecollection",
    "type": "CoverageCollection"
  }
}
```

**Recommendation:** The above `Link` header with `rel="collection"` should be included in coverage resources of
the collection to provide context and a way of navigation independent of what might be
stored inside the coverage format itself (see `"inCollection"`).

**Requirement:** If the coverage format does not support including a reference back to the collection
resource, then the above `Link` header with `rel="collection"` must be included.

Whether to include the full coverage data in a collection resource or just
provide links to it must be decided case-by-case.

**Recommendation:** The data volume of a resource should be kept to an amount that web clients
can still handle easily, which typically means up to a few megabytes (after gzip) at most.

### 4.1. Alternatives for unsuitable formats

If a coverage data format does not support collections or to split off coverages from a
collection, then the recommendations of this section can still be implemented by using a mix of formats. 
The only requirement is that the coverage data format must allow it to represent a single coverage
in a single resource. That is, it must not be a collection-only format.

**Recommendation:** If collections with split off coverages are not supported by the coverage data
format, then a different format should be used for collection resources.

For example, a set of GeoTIFF resources that are exposed as individual resources can be
grouped together with a collection resource in a different format.
The collection format could be using JSON-LD with the lightweight Hydra collection vocabulary.
An alternative to JSON-LD may be ATOM feeds. Selecting a suitable collection format depends on many factors,
some of which are:
- degree of support in clients and servers (e.g. XML vs JSON)
- customizability, e.g. ability to include coverage-related metadata in an interoperable way
- effort to add advanced functionality like paging or filtering (including self-describing API control data)

**Recommendation:** JSON-LD should be used as an alternative collection format.

## 5. Paged collection resources

Coverage data can become big quite quickly.
Therefore, different strategies are needed for clients to handle such data in a
convenient way. One such strategy is to offer paged collection resources.

**Recommendation:** If the coverage collection format supports or allows to integrate paging in a natural way,
then paged collection resources should be offered for collections with big numbers of coverages.

**Example serving a paged CoverageJSON collection:**
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
  "totalItems": 22021,
  "view" : {
    "id" : "#pagination",
    "@graph" : {
      "id" : "http://example.com/coveragecollection?page=1",
      "type" : "PartialCollectionView",
      "itemsPerPage": 100,
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

**Requirement:** A collection itself and a page of it must be separate resources,
including the first page.

**Requirement:** Each page resource must include links to navigate the paged collection,
at a minimum a link to the next page, if it is not the last.
If the format does not allow to include such links, then Link headers as shown above
with `rel="first"`, `rel="prev"`, `rel="next"`, `rel="last"` must be used.

**Requirement:** If the format supports resource identifiers (as above), then the collection elements
have to be associated to the collection resource and *not* the page resource.
Only the navigation links may be associated with the page resource.
If the format does not support resource identifiers, then a Link header with `ref="canonical"`
is required that links back to the collection resource.

**Recommendation:** If a collection offers pages, then the collection resource should
redirect to the first page with a "303 See Other" HTTP status.

**Recommendation:** If a collection would only contain a single page, then paged
collection resources should not be offered at all.

**Recommendation:** The `Link` headers (incl. "canonical") should be included in page resources to provide
context and a way of navigation independent of what the format includes.

**Recommendation:** If JSON-LD is used as a format, then the navigation controls should be included
as above using the Hydra ontology in a non-default graph. The default graph should
not be used to logically separate the actual data from the control data.

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
The recommended way to offer such functionality is described in the following subsection.

### 6.1. `Prefer` header method for embedding data

**Recommendation:** If a resource should support the optional embedding of certain data, then
the `Prefer` header defined in [RFC7240](https://tools.ietf.org/html/rfc7240) and
the [`include`](http://www.w3.org/TR/ldp/#prefer-parameters) parameter of the `return` preference
defined in [LDP](http://www.w3.org/TR/ldp/) should be used for that purpose.

**Example of asking a server to preferably embed certain data:**
```sh
$ curl http://example.com/coveragecollection

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://coveragejson.org/def#Domain>; rel="http://coverageapi.org/ns#canInclude"
Link: <http://coveragejson.org/def#Range>; rel="http://coverageapi.org/ns#canInclude"
Vary: Prefer

{... domain and range are not embedded by default ...}
```
```sh
$ curl http://example.com/coveragecollection -H "Accept: application/prs.coverage+json" \
  -H "Prefer: return=representation; include=\"http://coveragejson.org/def#Domain http://coveragejson.org/def#Range\""

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://coveragejson.org/def#Domain>; rel="http://coverageapi.org/ns#canInclude"
Link: <http://coveragejson.org/def#Range>; rel="http://coverageapi.org/ns#canInclude"
Vary: Prefer

{... domain and range are embedded now ...}
```

A standard way to advertise available `include` URIs to the client does not exist yet.
In the example above, a custom predicate `http://coverageapi.org/ns#canInclude` in a `Link` header is used for that purpose.

**Recommendation:** If a resource supports optional embedding via the `Prefer` header, then
the available `include` URIs should be included in `Link` headers with `rel="http://coverageapi.org/ns#canInclude"`.

**Requirement:** The `Prefer` header must not be more than a preference.
A server may not respect that preference and the client is expected to handle the situation regardless.

**Recommendation:** If a resource includes `Link` headers with `rel="http://coverageapi.org/ns#canInclude"`,
then the server should also fulfill those preferences if the client sends them within a request.

The client can inspect whether the server fulfilled a preference by looking at the returned coverage data.

Note that the above method requires a server implementation of 
[CORS "preflight"](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing#Preflight_example) requests
(HTTP OPTIONS) which browsers will send when using the `Prefer` header for cross-domain requests.

**Recommendation:** Browser clients that access cross-domain coverage data should only send a `Prefer` header
after they have confirmed that the server understands it. The client should assume that this is the case
if a `Link` header with `rel="http://coverageapi.org/ns#canInclude"` is included or the `Vary` header
includes `Prefer`. Furthermore, the client should assume that any related coverage data resource on the same
domain has the same support. The reason for confirming support before-hand is that not all servers
implement CORS "preflight" requests which would mean that in those cases the request would simply fail, but in
fact would succeed if the `Prefer` header had not been included.

If the above method using the `Prefer` header is not suitable for a specific API implementation,
then an alternative based on URL templates ([RFC6570](https://tools.ietf.org/html/rfc6570)) may be used
as described in the following.

### 6.2. URL template method for embedding data

**Example of asking a server to embed certain data with URL templates:** 
```sh
$ curl http://example.com/coveragecollection -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json

{
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org",
    {
      "api": "http://coverageapi.org/ns#api",
      "owl": "http://www.w3.org/2002/07/owl#",
      "DataRange": "owl:DataRange",
      "oneOf": {"@id": "owl:oneOf", "@container": "@list"},
      "multipleValues": "https://schema.org/multipleValues"
    }
  ],
  "id": "http://example.com/coveragecollection",
  "type": "CoverageCollection",
  "coverages": [...],
  "api": {
    "id" : "#api",
    "@graph" : {
      "type": "IriTemplate",
      "template": "http://example.com/coveragecollection{?include*}",
      "mapping": [{
        "type": "IriTemplateMapping",
        "variable": "include",
        "property": {
          "id": "http://coverageapi.org/ns#preferInclude",
          "comment": "Indicates a preference that certain coverage data should be directly included in the resource. Currently, either 'domain' or 'range'.",
          "range": {
            "type": "DataRange",
            "oneOf": ["domain", "range"]
          }
        },
        "multipleValues": true,
        "required": false
      }]
    }
  }
}
```
```sh
$ curl http://example.com/coveragecollection?include=domain&include=range \
  -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://example.com/coveragecollection>; rel="canonical"

{
  "id": "http://example.com/coveragecollection",
  ... identical to above but with domains and ranges embedded ...
}
```

**Requirement:** If a server decides to reject a request for embedding data based on URL templates,
then it must redirect with a "303 See Other" HTTP status to a resource whose preferences the server can fulfill.

**Example of rejecting a request for including data:**
```sh
$ curl http://example.com/coveragecollection?include=domain&include=range \
  -H "Accept: application/prs.coverage+json"

HTTP/1.1 303 See Other
Location: http://example.com/coveragecollection
Content-length: 0
```

**Requirement:** If embedding data is supported via URL templates then those templates must be included
in the coverage data format in an interoperable way.

**Requirement:** If the format supports resource identifiers (as above), then the collection elements
have to be associated to the collection resource and *not* the resource that corresponds to the URL
template for embedding data.
If the format does not support resource identifiers, then a Link header with `rel="canonical"`
is required that links back to the collection resource.

**Recommendation:** If JSON-LD is used as a format, then the URL template for requesting to
embed data should be included as above using the Hydra ontology in a non-default graph. The default graph should
not be used to logically separate the actual data from the control data.

**Recommendation:** The `Link` header with `rel="canonical"` should be included in resources
that correspond to the URL template for embedding data to provide context independent of what the format includes.

Note that this specification does *not* force a specific URL template. The important detail
is only the `"property"` within the template mapping, which is `http://coverageapi.org/ns#preferInclude`
and which should be used in non-JSON-LD scenarios as well if possible.
The property should be the only trigger for clients to know what the parameter means and how to use it. 

NOTE: "multipleValues" is [not standardized in Hydra](https://lists.w3.org/Archives/Public/public-hydra/2015Nov/0082.html) yet.

## 7. Spatiotemporally filtered collection resources

A common use case is to filter big coverage collections by a certain geographical area or
time period. The recommended way to offer such functionality is described below.

**Example of spatiotemporally filtering a collection:**
```sh
$ curl http://example.com/coveragecollection -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json

{
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org",
    {
      "api": "http://coverageapi.org/ns#api",
      "opensearchgeo": "http://a9.com/-/opensearch/extensions/geo/1.0/",
      "opensearchtime": "http://a9.com/-/opensearch/extensions/time/1.0/"
    }
  ],
  "id": "http://example.com/coveragecollection",
  "type": "CoverageCollection",
  "coverages": [...],
  "api": {
    "id" : "#api",
    "@graph" : {
      "type": "IriTemplate",
      "template": "http://example.com/coveragecollection{?bbox,timeStart,timeEnd}",
      "mapping": [{
        "type": "IriTemplateMapping",
        "variable": "bbox",
        "property": {
          "id": "opensearchgeo:box",
          "comment": "The box is defined by 'west, south, east, north' coordinates of longitude, latitude, in EPSG:4326 decimal degrees. For values crossing the 180 degrees meridian the west value should be bigger than the east value.",
          "range": "xsd:string"
        },
        "required": false
      }, {
        "type": "IriTemplateMapping",
        "variable": "timeStart",
        "property": {
          "id": "opensearchtime:start",
          "comment": "Character string with the start of the temporal interval according to an RFC3339 date-time.",
          "range": "xsd:string"
        },
        "required": false
      }, {
        "type": "IriTemplateMapping",
        "variable": "timeEnd",
        "property": {
          "id": "opensearchtime:end",
          "comment": "Character string with the end of the temporal interval according to an RFC3339 date-time.",
          "range": "xsd:string"
        },
        "required": false
      }]
    }
  }
}
```
```sh
$ curl http://example.com/coveragecollection?bbox=120,10,134,14&timeStart=2012-01-01T00:00:00Z&timeEnd=2012-02-01T00:00:00Z \
  -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json

{
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org",
    {
      "api": "http://coverageapi.org/ns#api",
      "filteredFrom": "http://purl.org/dc/terms/source",
      "opensearchgeo": "http://a9.com/-/opensearch/extensions/geo/1.0/",
      "opensearchtime": "http://a9.com/-/opensearch/extensions/time/1.0/"
    }
  ],
  "id": "http://example.com/coveragecollection?bbox=120,10,134,14&timeStart=2012-01-01T00:00:00Z&timeEnd=2012-02-01T00:00:00Z",
  "type": "CoverageCollection",
  "coverages": [...those that match the filter...],
  "filteredFrom": {
    "type": "CoverageCollection",
    "id": "http://example.com/coveragecollection",
    "api": {...as above...}
  }
}
```

The example reuses URL parameters that are defined in the [OpenSearch Geo & Time standard](http://www.opengis.net/doc/IS/opensearchgeo/1.0).
This standard describes the exact syntax and semantics of the parameters together with additional parameters that can be used, for example,
`opensearchgeo:relation` which may be one of "intersects", "contains", or "disjoint" with the default being "intersects".

**Requirement:** If a collection resource supports spatiotemporal filtering, then the corresponding URL templates
must be included in the coverage data format in an interoperable way.

**Recommendation:** If the format supports resource identifiers (as above), then the collection elements
should be associated to the filtered collection resource and not the unfiltered collection resource.
That way it is straight-forward to add paging functionality and a `"totalItems"` count to the filtered
collection.
A link to the unfiltered collection resource should be included within the filtered collection.

**Recommendation:** If JSON-LD is used as a format, then the URL template for filtering the collection
should be included as above using the Hydra ontology in a non-default graph. The default graph should
not be used to logically separate the actual data from the control data.

**Requirement:** If a server cannot fulfill the requested filtering operation for some reason, e.g. because
the client used a wrong parameter syntax, then an error response with a suitable
HTTP status code like `400 Bad Request` must be returned.

**Recommendation:** An error response should include details on the error and be in a format
that the client can easily understand. In particular "[Problem Details for HTTP APIs](https://github.com/dret/I-D/tree/master/http-problem-rdf)"
should be returned for non-HTML requests instead of inventing a custom error format.

**Example of requesting a resource with invalid filter parameters:**
```sh
$ curl http://example.com/coveragecollection?timeStart=yesterday&timeEnd=today \
  -H "Accept: application/prs.coverage+json"

HTTP/1.1 400 Bad Request
Content-Type: application/ld+json

{
  "@context": "https://raw.githubusercontent.com/dret/I-D/master/http-problem-rdf/http-problem-context.jsonld",
  "type": "http://example.com/problems#InvalidSyntax",
  "title": "Invalid query parameter syntax.",
  "detail": "Invalid syntax for timeStart and timeEnd query parameters, must be in RFC3339 date-time format."
}
```

Note that if more filtering parameters are required than are defined within OpenSearch Geo & Time
(for example, depth/vertical filtering), then a new template variable from another such standard
may be used, or a new custom one created if none exists, under a different URI namespace.

Note that equally to the previous section this specification does *not* force a specific URL template.

## 8. Spatiotemporally subsetted resources


**Example of subsetting a single coverage:**
```sh
$ curl http://example.com/coveragecollection/coverage1 -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://example.com/coveragecollection>; rel="collection"

{
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org",
    {
      "covapi": "http://coverageapi.org/ns#",
      "api": "covapi:api",
      "opensearchgeo": "http://a9.com/-/opensearch/extensions/geo/1.0/",
      "opensearchtime": "http://a9.com/-/opensearch/extensions/time/1.0/",
      "inCollection": { "@reverse": "hydra:member" }
    }
  ],
  "id": "http://example.com/coveragecollection/coverage1",
  "type": "GridCoverage",
  "title": "First coverage",
  "domain": {...},
  "ranges": {...},
  "inCollection": {
    "id": "http://example.com/coveragecollection",
    "type": "CoverageCollection"
  },
  "api": {
    "id" : "#api",
    "@graph" : {
      "type": "IriTemplate",
      "template": "http://example.com/coveragecollection/coverage1{?subsetBbox,subsetTimeStart,subsetTimeEnd}",
      "mapping": [{
        "type": "IriTemplateMapping",
        "variable": "subsetBbox",
        "property": {
          "id": "covapi:subsetBbox",
          "comment": "The box is defined by 'west, south, east, north' coordinates of longitude, latitude, in EPSG:4326 decimal degrees. For values crossing the 180 degrees meridian the west value should be bigger than the east value.",
          "range": "opensearchgeo:box"
        },
        "required": false
      }, {
        "type": "IriTemplateMapping",
        "variable": "subsetTimeStart",
        "property": {
          "id": "covapi:subsetTimeStart",
          "comment": "Character string with the start of the temporal interval according to RFC3339.",
          "range": "opensearchtime:start"
        },
        "required": false
      }, {
        "type": "IriTemplateMapping",
        "variable": "subsetTimeEnd",
        "property": {
          "id": "covapi:subsetTimeEnd",
          "comment": "Character string with the end of the temporal interval according to RFC3339.",
          "range": "opensearchtime:end"
        },
        "required": false
      }]
    }
  }
}
```
```sh
$ curl http://example.com/coveragecollection/coverage1?subsetBbox=120,10,134,14&subsetTimeStart=2015-01-01T00:00:00Z \ 
  -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://example.com/coveragecollection?subsetBbox=120,10,134,14&subsetTimeStart=2015-01-01T00:00:00Z>; rel="collection"

{
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org",
    {
      "covapi": "http://coverageapi.org/ns#",
      "api": "covapi:api",
      "subsetOf": "http://purl.org/dc/terms/source",
      "inCollection": { "@reverse": "hydra:member" },
      "opensearchgeo": "http://a9.com/-/opensearch/extensions/geo/1.0/",
      "opensearchtime": "http://a9.com/-/opensearch/extensions/time/1.0/"
    }
  ],
  "id": "http://example.com/coveragecollection/coverage1?subsetBbox=120,10,134,14&subsetTimeStart=2015-01-01T00:00:00Z",
  "type": "GridCoverage",
  "title": "First coverage",
  "domain": {... subsetted ...},
  "ranges": {... subsetted ...},
  "inCollection": {
    "id": "http://example.com/coveragecollection?subsetBbox=120,10,134,14&subsetTimeStart=2015-01-01T00:00:00Z",
    "type": "CoverageCollection",
    "subsetOf": {
      "id": "http://example.com/coveragecollection",
      "type": "CoverageCollection"
    }
  },
  "subsetOf": {
  	"id": "http://example.com/coveragecollection/coverage1",
  	"type": "GridCoverage",
    "api": {
      "id" : "#api",
      "@graph" : {
        "type": "IriTemplate",
        "template": "http://example.com/coveragecollection/coverage1{?subsetBbox,subsetTimeStart,subsetTimeEnd}",
        "mapping": [...]
      }
    }
  }
}
```

Note in the last response that the API metadata is *not* defined for the subsetted resource
but instead for the original coverage.
By doing that, the server states that the subsetted resource cannot be directly further subsetted,
but instead any further subsetting has to begin at the original coverage.

In the example above, subsetting is done on a single coverage. The same technique can also be
applied to coverage collections, as hinted in the `"inCollection"` field of the subsetted coverage.

Note that equally to the previous section this specification does *not* force a specific URL template.

## 9. Index-based subsetted resources

Subsetting coverage data by defining coordinate bounds (previous section) is not always desirable.
When a domain axis shall be fixed to a single coordinate (e.g. a given time or height step),
then it may be easier to operate in the index space of the coverage domain.
This is typically only possible if the corresponding coverage domain is already known.

**Example of subsetting a single coverage:**
```sh
$ curl http://example.com/coveragecollection/coverage1 -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json
Link: <http://example.com/coveragecollection>; rel="collection"

{
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org",
    {
      "covapi": "http://coverageapi.org/ns#",
      "api": "covapi:api",
      "opensearchgeo": "http://a9.com/-/opensearch/extensions/geo/1.0/",
      "opensearchtime": "http://a9.com/-/opensearch/extensions/time/1.0/",
      "inCollection": { "@reverse": "member" },
      "multipleValues": "https://schema.org/multipleValues"
    }
  ],
  "id": "http://example.com/coveragecollection/coverage1",
  "type": "GridCoverage",
  "title": "First coverage",
  "domain": {...},
  "ranges": {...},
  "inCollection": {
    "id": "http://example.com/coveragecollection",
    "type": "CoverageCollection"
  },
  "api": {
    "id" : "#api",
    "@graph" : {
      "type": "IriTemplate",
      "template": "http://example.com/coveragecollection/coverage1{?subsetIndex*}",
      "mapping": [{
        "type": "IriTemplateMapping",
        "variable": "subsetIndex",
        "property": {
          "id": "covapi:subsetIndex",
          "comment": "numpy-style slicing syntax: 'x[start:stop:step]' or 'x[start:stop]' or 'x[i]' or 'x[i,j]' or 'x[i,j,k]' etc. where 'x' is the axis alias, 0 <= 'start' < 'stop', 'step' >= 1 (default 1), and 0 <= 'i','j','k'",
          "range": "xsd:string"
        },
        "multipleValues": true,
        "required": false
      }]
    }
  }
}
```
```sh
$ curl http://example.com/coveragecollection/coverage1?subsetIndex=x[0:10]&subsetIndex=t[10] \
  -H "Accept: application/prs.coverage+json"

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json

{
  "@context": [
    "http://www.w3.org/ns/hydra/core",
    "http://coveragejson.org",
    {
      "covapi": "http://coverageapi.org/ns#",
      "api": "covapi:api",
      "subsetOf": "http://purl.org/dc/terms/source",
      "inCollection": { "@reverse": "member" },
      "opensearchgeo": "http://a9.com/-/opensearch/extensions/geo/1.0/",
      "opensearchtime": "http://a9.com/-/opensearch/extensions/time/1.0/"
    }
  ],
  "id": "http://example.com/coveragecollection/coverage1?subsetIndex=x[0:10]&subsetIndex=t[10]",
  "type": "GridCoverage",
  "title": "First coverage",
  "domain": {... subsetted ...},
  "ranges": {... subsetted ...},
  "subsetOf": {
  	"id": "http://example.com/coveragecollection/coverage1",
  	"type": "GridCoverage",
    "inCollection": {
      "id": "http://example.com/coveragecollection",
      "type": "CoverageCollection",
    },
    "api": {
      "id" : "#api",
      "@graph" : {
        "type": "IriTemplate",
        "template": "http://example.com/coveragecollection/coverage1{?subsetIndex*}",
        "mapping": [...]
      }
    }
  }
}
```

Note that in the above response the subsetted coverage is not directly associated to a subsetted
collection, contrarily to the previous section when subsetting by coordinates.
The reason is that a collection can typically not be subsetted as a whole by axis indices,
since not all coverages of the collection may have exactly the same domain geometry.
Instead, the relation to the parent collection is established via the parent coverage (see `"subsetOf"`). 

Note that equally to the previous section this specification does *not* force a specific URL template.
For example, the template could also be `http://example.com/coverage1?subset={subsetIndex*}`
which corresponds to URLs like `http://example.com/coverage1?subset=x[0:10],y[0]`
