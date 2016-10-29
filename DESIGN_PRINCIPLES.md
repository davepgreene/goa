# The Principles Behind the DSL of goa v2

## `API` Expression

Like in v1 the top level DSL function in v2 is `API`. The `API` DSL lists the
global properties of the API such as its hostname, its version number etc.

```go
var _ = API("cellar", func() {
    Title("The virtual wine cellar")
    Version("1.0")
    Description("An example of an API implemented with goa")
    Contact(func() {
        Name("goa team")
        Email("admin@goa.design")
        URL("http://goa.design")
                })
    License(func() {
        Name("MIT")
    })
    Docs(func() {
        Description("goa guide")
        URL("http://goa.design/getting-started.html")
    })
    Host("cellar.goa.design")
})
```

## `Service` Expression

The `Service` DSL defines a group of endpoints. This maps to a resource in REST
or a `service` declaration in gRPC. A service may define a default type (with
`DefaultType`). The default type lists common attributes that may be reused
throughout the service endpoint request and response types. A service may also
define common error responses to all the service endpoint, more on error
responses in the next section.

```go
// The "account" service exposes the account resource endpoints.
var _ = Service("account", func() {
    DefaultType(Account)
    Error(ErrUnauthorized, Unauthorized)
    HTTP(func() {
        BasePath("/accounts")
    })
    GRPC(func() {
        Name("Account")
    })
```

The `HTTP` and `GRPC` functions make it possible to define transport specific
service properties such as a common base path to all HTTP requests or the GRPC
service name.

## `Endpoint` Expression

The service endpoints are described using `Endpoint`. This function defines the
endpoint request and response types. It may also list an arbitrary number of
error responses. An error response has a name and optionally a type. If the
`Endpoint` DSL omits the response type then the service default type is used
instead. The built-in type `Empty` denotes an empty response (no response body
in HTTP, Empty message in gRPC).

```go
    Endpoint("update", func() {
        Description("Change account name")
        Request(UpdateAccount)
        Response(Empty)
        Error(ErrNotFound)
        Error(ErrBadRequest, ErrorResponse)
```

The request, response and error types define the request and responses
*independently of the transport*. The `HTTP` function defines the mapping of
request and response type attributes to the HTTP request path and query string
values as well as the HTTP request and response headers and bodies.
The `HTTP` function also defines other HTTP specific properties such as the
request path, the response HTTP status codes etc.
The `GRPC` function indicates the name of the gRPC endpoint and any gRPC
options. It also defines how errors are handled by specifying the name of
response attributes used to store the error message (defaults to the name of the
error). If no attribute is specified and the name of the error does not match
one then the error is returned as a gRPC error rather than embedded in the
response type.

```go
        HTTP(func() {
            PUT("/{accountID}")    // "accountID" request attribute
            Body(func() {
                Attribute("name")  // "name" request attribute
                Required("name")
            })
            Response(NoContent)
            Error(ErrNotFound, NotFound)
            Error(ErrBadRequest, BadRequest, ErrorResponse)
        })
        GRPC(func() {
            Name("Update")
            Error(ErrNotFound, func() {
                Field("Err")
            })
        })
```

### Endpoint Request Type

In the example above the `accountID` HTTP request path parameter is defined by
the attribute of the `UpdateAccount` type with the same name and so is the body
attribute `name`.

Any attribute that is no explicitly mapped by the `HTTP` function is implicitly
mapped to request body attributes. This makes is simple to define mappings where
only one of the fields for the request type is mapped to the header and all
other fields are mapped tp the body.

The body attributes may also be listed explicitly using the `Body` function.
This function accepts either a DSL listing the body attributes or the name of a
request type attribute whose type defines the body as a whole. The latter makes
it possible to use any arbitrary type to describe request body and not just
object, for example the attribute (and thus the body) could be an array.

Implicit request body definition:

```go
        HTTP(func() {
            PUT("/{accountID}")    // "accountID" request attribute
            Response(NoContent)
            Error(ErrNotFound, NotFound)
            Error(ErrBadRequest, BadRequest, ErrorResponse)
        })
```

Array body definition:

```go
        HTTP(func() {
            PUT("/")
            Body("names") // Assumes request type has attribute "names"
            Response(NoContent)
            Error(ErrNotFound, NotFound)
            Error(ErrBadRequest, BadRequest, ErrorResponse)
        })
```

### Endpoint Response Type

While a service may only define one response type the `HTTP` function may list
multiple responses. Each response defines the HTTP status code, response body
shape (if any) and may also list HTTP headers.

By default the shape of the body of responses with HTTP status code 200 is
described by the endpoint response type.  The `HTTP` function may optionnally
use response type attributes to define response headers. Any attribute of the
response type that is not explicitly used to define a response header defines a
field of the response body implcitly. This alleviates the need to repeat all the
response type attributes to define the body since in most cases only a few would
map to headers.

The response body may also be explicitly described using the function `Body`.
The function works identically as when used to describe the request body: it may
be given a list of response type attributes in which case the body shape is an
object or the name of a specific attribute in which case the response body shape
is dictated by the type of the attribute.

```go
    Endpoint("index", func() {
        Description("Index all accounts")
        Request(ListAccounts)
        Response(func() {
            Attribute("marker", String, "Pagination marker")
            Attribute("accounts", CollectionOf(Account), "list of accounts")
        })
        HTTP(func() {
            GET("")
            Response(OK, func() {
                Header("marker")
                Body("accounts")
            })
        })
    })
```

The example produces response bodies of the form
`[{"name"="foo"},{"name"="bar"}]` assuming the type `Account` only has a `name`
attribute. The same example as above but with the line defining the response
body (`Body("accounts")`) removed produces response bodies of the form:
`{"accounts":[{"name"="foo"},{"name"="bar"}]` since `accounts` isn't used
to define headers.

## Data Types

Like in v1, the built-in types are primitive types, array, map and object types
(note the change of nomenclature and DSL from `hash` to `map`).

The list of primitive types in v2 is:

* `Int32`, `Int64`, `UInt32`, `UInt64`
* `Float32`, `Float64`
* `String`, `Bytes`
* `Any` (maps to any type, primitive or not)

Like in v1 arrays can be declared in one of two ways:

* `ArrayOf()` which accepts any type or media type and returns a type
* `CollectionOf()` which accepts media types only and returns a media type

The media type returned by `CollectionOf` contains the same views as the media
type given as argument. Each view simply renders an array where each element has
been projected using the corresponding element view. The media type id of the
collection is computed by appending the `;collection` qualifier to the element
media type id.

Like in v1 the goa DSL makes it possible to define both user and media types.
Media types are user types that also define a media type id, views and links.
The DSL for defining user types and media types is the same as in v1 (using
`Type` and `MediaType` respectively).

### gRPC: Attribute Tags

gRPC (and other RPC protocols) requires that each attribute defined on a type or
media type be tagged with a unique integer. This tag is used to pack the data on
the wire and must thus never change as the type evolves. It is therefore
necessary to explicitly defines the tags, they cannot be simply inferred using
the position of the attribute for example.

There are two ways a tag may be defined in the DSL: using metadata or using the
new `Field` function. Using metadata simply consists of adding the tag metadata
to the attribute, for example:

```go
    Attribute("A", Int32, func() {
        Metadata("rpc:tag", "1")
    })
```

The `Field` function is syntactic sugar that does the exact same thing as above
and accepts the tag as first argument:

```go
    Field(1, "A", Int32)
```

Types defined using `Field` instead of `Attributes` can be used to define HTTP
endpoints, the metadata is simply isgnored by the generators in this case.

### gRPC: Using Protobuf Files

So far we have seen how to describe gRPC services using the goa DSL. The
`goagen` tool uses that information to generate a protobuf file and then invokes
the `protoc` compiler to generate the corresponding code:

```
goa DSL --goagen--> api.proto --protoc--> *.go
```

Often times there may already exist a protobuf definition file for a gRPC
service. Protobuf focuses on making it possible to describe the information
required to represent the data on the wire and the service endpoints. The goa
DSL also makes it possible to provide documentation, validations, default values
etc. The idea is thus to make it possible to "point" to the relevant part of a
protobuf file from the goa DSL while still allowing for describing the extra
information.

Concretely the new `Proto` function accepts the name of a package and an
identifier to one of the service or type declarations in it. The function may be
used when defining services or types in the goa DSL:

```go
var _ = Service("manager", func() {
    Description("Exposes endpoints to manage accounts")
    // Use rpc endpoints defined in the Manager service of package cellar.api
    Proto("cellar.api", "Manager")
})

var _ = Type("account", func() {
    Description("The account request type")
    // Use message "Account" of package cellar.api
    Proto("cellar.api", "Account")
})
```

