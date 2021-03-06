# Coverage Data REST API Core Specification

This specification is an attempt to provide the basics for developing a modern and
interoperable REST API for accessing [coverage data](https://en.wikipedia.org/wiki/Coverage_data)
such as earth observation imagery. 
Its focus is not on dataset metadata or cataloguing of such, but techniques
of accessing coverage data of a dataset once that dataset has been discovered
by other means.
It may be considered as a modern alternative to [OGC](http://www.opengeospatial.org/)'s
[Web Coverage Service](https://en.wikipedia.org/wiki/Web_Coverage_Service),
however, it is important to understand that this specification is not a single ready-to-use
API but rather a modular blueprint for developing such interoperable APIs.

This specification is based on the following main principles:
- *modular* - implement only what you or your users need
- *self-describing* - advertise API controls as part of each resource
- *customizable* - no fixed URL structures
- *extensible* - add your own API features
- *format-agnostic* - independent of concrete coverage data formats

This document should be considered as work-in-progress and may change at any time.

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
a collection of supported concepts, it is more important to refer to and uniquely identify
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
then [content negotiation](https://en.wikipedia.org/wiki/Content_negotiation) should be used,
while still offering each format through a unique URL and establishing links between those formats via
`Link` headers.

Content negotiation requires more support on the server side but makes sure that the
Coverage Data as a concept stays [at a *single* URL](https://www.w3.org/TR/dwbp/#DataIdentifiers) (which is important
in the linked web).

**Example using content negotiation without redirects:**
```sh
$ curl http://example.com/coveragedata -H "Accept: application/x-netcdf"

HTTP/1.1 200 OK
Content-Type: application/x-netcdf
Content-Location: http://example.com/coveragedata.nc
Vary: Accept
Link: <http://example.com/coveragedata.geotiff>; rel="alternate"; type="image/tiff"

[binary netcdf]
```
```sh
$ curl http://example.com/coveragedata -H "Accept: image/tiff"

HTTP/1.1 200 OK
Content-Type: image/tiff
Content-Location: http://example.com/coveragedata.geotiff
Vary: Accept
Link: <http://example.com/coveragedata.nc>; rel="alternate"; type="application/x-netcdf"

[binary geotiff]
```

A common alternative to directly serving data in a specific format is to redirect to a static resource that does not support content negotiation.

**Example using content negotiation with redirects:**
```sh
$ curl http://example.com/coveragedata -H "Accept: application/x-netcdf"

HTTP/1.1 303 See Other
Location: http://example.com/coveragedata.nc
Vary: Accept
Content-length: 0
```
```sh
$ curl http://example.com/coveragedata.nc

HTTP/1.1 200 OK
Content-Type: application/x-netcdf
Link: <http://example.com/coveragedata.geotiff>; rel="alternate"; type="image/tiff"

[binary netcdf]
```

The idea is that the client is aware of the formats it supports and therefore
knows the media types to include in the "Accept" header of the first request.
This is how web browsers work as well, they know HTML, so they
specifically request that format.
In addition, the `Link` header points to URLs of alternative formats. 

**Pro Tip:** Offer HTML as format as well so that *humans* can explore the coverage data as well.

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
are interested in (since the collection resource might only include overview data by default then).
Note that this does not mean that there can't be a resource or representation which
offers the collection with full coverage data embedded (for example, CoverageJSON uses the "profile" media type
parameter with which an embedded-data version can be explicitly requested).

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
    "type": "Coverage",
    "title": "First coverage",
    "domain": "http://example.com/coverage1_domain",
    "ranges": {"TEMP": "http://example.com/coveragecollection/coverage1/TEMP"} 
  }, {
    "id": "http://example.com/coveragecollection/coverage2",
    "type": "Coverage",
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
  "type": "Coverage",
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

**Recommendation:** If a collection offers pages, then the collection resource should
redirect to the first page with a "303 See Other" HTTP status.

**Recommendation:** If a collection would only contain a single page, then paged
collection resources should not be offered at all.

**Recommendation:** The `Link` headers should be included in page resources to provide
context and a way of navigation independent of what the format includes.

**Recommendation:** If JSON-LD is used as a format, then the navigation controls should be included
as above using the Hydra ontology in a non-default graph. The default graph should
not be used to logically separate the actual data from the control data.

## 6. Spatiotemporally filtered collection resources

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

## 7. Spatiotemporally subsetted resources


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

## 8. Index-based subsetted resources

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
          "comment": "numpy-style slicing syntax: 'x[start:stop:step]' or 'x[start:stop]' or 'x[i]'. where 'x' is the axis alias, 0 <= 'start' < 'stop', 'step' >= 1 (default 1), and 0 <= 'i'",
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
