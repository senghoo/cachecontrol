# cachecontrol: HTTP Caching Parser and Interpretation

[![GoDoc][1]][2][![Build Status](https://travis-ci.org/pquerna/cachecontrol.svg?branch=master)](https://travis-ci.org/pquerna/cachecontrol)
[1]: https://godoc.org/github.com/pquerna/cachecontrol?status.svg
[2]: https://godoc.org/github.com/pquerna/cachecontrol
 

`cachecontrol` implements [RFC 7234](http://tools.ietf.org/html/rfc7234) __Hypertext Transfer Protocol (HTTP/1.1): Caching__.  It does this by parsing the `Cache-Control` and other headers, providing information about requests and responses -- but `cachecontrol` does not implement an actual cache backend, just the control plane to make decisions about if a particular response is cachable.

# Usage

## Can you cache Example.com?

```go
package main

import (
	"github.com/pquerna/cachecontrol"

	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {
	req, _ := http.NewRequest("GET", "http://www.example.com/", nil)

	res, _ := http.DefaultClient.Do(req)
	_, _ = ioutil.ReadAll(res.Body)

	reasons, expires, _ := cachecontrol.CachableResponse(req, res, cachecontrol.Options{})

	fmt.Println("Reasons to not cache: ", reasons)
	fmt.Println("Expiration: ", expires.String())
}
```

## Can I use this in a high performance caching server?

`cachecontrol` is divided into two packages: `cachecontrol` with a high level API, and a lower level `cacheobject` package.  Use `[Object](https://godoc.org/github.com/pquerna/cachecontrol/cacheobject#Object)` in a high performance use case where you have previously parsed headers containing dates or would like to avoid memory allocations.

```go
package main

import (
	"github.com/pquerna/cachecontrol/cacheobject"

	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {
	req, _ := http.NewRequest("GET", "http://www.example.com/", nil)

	res, _ := http.DefaultClient.Do(req)
	_, _ = ioutil.ReadAll(res.Body)

	reqDir, _ := cacheobject.ParseRequestCacheControl(req.Header.Get("Cache-Control"))

	resDir, _ := cacheobject.ParseResponseCacheControl(res.Header.Get("Cache-Control"))
	expiresHeader, _ := http.ParseTime(res.Header.Get("Expires"))
	dateHeader, _ := http.ParseTime(res.Header.Get("Date"))
	lastModifiedHeader, _ := http.ParseTime(res.Header.Get("Last-Modified"))

	obj := cacheobject.Object{
		RespDirectives:         resDir,
		RespHeaders:            res.Header,
		RespStatusCode:         res.StatusCode,
		RespExpiresHeader:      expiresHeader,
		RespDateHeader:         dateHeader,
		RespLastModifiedHeader: lastModifiedHeader,

		ReqDirectives: reqDir,
		ReqHeaders:    req.Header,
		ReqMethod:     req.Method,
	}
	rv := cacheobject.ObjectResults{}

	cacheobject.CachableObject(&obj, &rv)
	cacheobject.ExpirationObject(&obj, &rv)

	fmt.Println("Errors: ", rv.OutErr)
	fmt.Println("Reasons to not cache: ", rv.OutReasons)
	fmt.Println("Warning headers to add: ", rv.OutWarnings)
	fmt.Println("Expiration: ", rv.OutExpirationTime.String())
}
```

# License

[Apache 2.0](./LICENSE)
