# Coverage Data REST API Specification

## 1. Introduction

In the following, the term "Coverage Data" stands both for a single Coverage,
but also for a collection of Coverages.
A Coverage can contain multiple measured/computed parameters, like wind speed
and wind direction.

As an example, a netCDF file (extension `.nc`) can contain a single or multiple
coverages. The same is true for a CoverageJSON file.
A GeoTIFF file contains a single coverage.

### Notes

In example responses, HTTP headers that are not relevant may be omitted.

## 2. Static resources

In its simplest form, Coverage Data is made available as one or more resources,
served with the correct media type. The resources can be static files.

Example serving a netCDF file:
```
$ curl http://example.com/coveragedata.nc

HTTP/1.1 200 OK
Content-Type: application/x-netcdf

[binary]
```

Example serving a GeoTIFF file:
```
$ curl http://example.com/coveragedata.geotiff

HTTP/1.1 200 OK
Content-Type: image/tiff

[binary]
```

## 3. Content negotiation

If the same Coverage Data should be made available in different formats,
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

[binary]
```
```
$ curl -H "Accept: image/tiff" http://example.com/coveragedata

HTTP/1.1 200 OK
Content-Type: image/tiff

[binary]
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
(see the "Simple resources" section), which then would always deliver the corresponding format.

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

If possible, this separation should be exploited, instead of serving a single file
with all coverages fully embedded. The advantage is that having separate resources for each
coverage allows to link to them, and also allows clients to load only the coverages they
are interested in. Note that this does not mean that there can't be a resource which
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

## 5. Paged collection resources

Offering paged collection resources is one of several optional capabilities that can be added to the API.


