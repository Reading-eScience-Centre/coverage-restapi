# Coverage Data REST API Core Specification

## 1. Introduction

This API specification is structured in such a way that a server implementation
can evolve naturally, beginning with no custom implementation at all.
It is based on largely independent concepts of which some or all can be integrated,
depending on which level of sophistication is necessary, which in turn is
determined by the amount of data to be served, the type of users, and their access patterns.

It is important to realize that there is no requirement to implement one or the other
aspect detailed below. This is made possible by self-describing and therefore
advertising the offered capabilities that the server offers in each and every resource,
without relying on a fixed structure like an API root metadata document.

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
```
$ curl http://example.com/coveragedata.nc

HTTP/1.1 200 OK
Content-Type: application/x-netcdf

[binary netcdf]
```

Example serving a GeoTIFF file:
```
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
```
$ curl -H "Accept: application/x-netcdf" http://example.com/coveragedata

HTTP/1.1 200 OK
Content-Type: application/x-netcdf

[binary netcdf]
```
```
$ curl -H "Accept: image/tiff" http://example.com/coveragedata

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
```
$ curl http://example.com/coveragecollection.covjson

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json

{
  "type": "CoverageCollection",
  "parameters": {...},
  "coverages": [{
    "type": "GridCoverage",
  	"id": "http://example.com/coverage1.covjson",
    "title": "First coverage",
    "domain": "http://example.com/coverage1_domain.covjson",
    "ranges": {"TEMP": "http://example.com/coverage1_TEMP.covjson"} 
  }, {
    "type": "GridCoverage",
  	"id": "http://example.com/coverage2.covjson",
    "title": "Second coverage",
    "domain": "http://example.com/coverage2_domain.covjson",
    "ranges": {"SPEED": "http://example.com/coverage2_SPEED.covjson"}
  }]
}
```
```
$ curl http://example.com/coverage1.covjson

HTTP/1.1 200 OK
Content-Type: application/prs.coverage+json

{
  "type": "GridCoverage",
  "id": "http://example.com/coverage1.covjson",
  "title": "First coverage",
  "domain": {...},
  "ranges": {"TEMP": {...}} 
}
```

Whether to include the full coverage data in a collection resource or just
provide links to it must be decided case-by-case.
It is advisable that the data volume is kept to an amount that web clients
can still handle easily, which typically means in the range of a few megabytes
at most.

## 5. Paged collection resources

Offering paged collection resources is one of several optional capabilities that can be added to the API.


