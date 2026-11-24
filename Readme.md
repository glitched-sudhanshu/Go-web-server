# Go HTTP Server: Understanding Form Submission

This project demonstrates how an HTML form communicates with a Go HTTP
server using HTTP.

## Project architecture
<img width="1140" height="563" alt="image" src="https://github.com/user-attachments/assets/1583aa36-5299-4643-aaf0-26a91f89ebcf" />

## Project structure

``` text
project/
├── main.go
└── static/
    ├── index.html
    └── form.html
```

## Run the application

From the project directory:

``` bash
go run main.go
```

Open `http://localhost:8080/` in your browser. The server serves static
files from `./static`.

## How form submission works

The HTML form specifies a request method and destination:

``` html
<form method="POST" action="/form">
```

-   `method="POST"` tells the browser to send a POST request.
-   `action="/form"` tells it to send the request to `/form` on the
    current origin.

When the user clicks Submit, the browser serializes the form fields
(identified by their `name` attributes) and sends an HTTP request. For
example:

``` http
POST /form HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

name=Sudhanshu&address=Delhi
```

The browser does not directly call a Go function.

### Request lifecycle

1.  The user clicks Submit.
2.  The browser reads the form fields and their `name` attributes.
3.  The browser encodes the values and sends an HTTP POST request to
    `/form`.
4.  The Go HTTP server, already listening on port `8080`, receives the
    request.
5.  Go's HTTP multiplexer matches `/form` to the registered
    `formHandler`.
6.  The server invokes `formHandler(w, r)`.
7.  The handler parses the form values and writes an HTTP response.
8.  The browser receives and displays the response.

``` text
Browser                         Go HTTP server
   |                                   |
   | POST /form                         |
   | name=Sudhanshu&address=Delhi       |
   |---------------------------------->|
   |                         DefaultServeMux
   |                         matches /form
   |                                   |
   |                         formHandler(w, r)
   |                         Parse form values
   |                         Write response
   |                                   |
   | HTTP response                     |
   |<----------------------------------|
   | Display response                  |
```

## How Go routes the request

In `main.go`, routes are registered using `http.Handle()` and
`http.HandleFunc()`. These registrations associate URL patterns with
handlers.

  URL pattern   Handler          Purpose
  ------------- ---------------- ----------------------------
  `/`           File server      Serves static files
  `/form`       `formHandler`    Processes form submissions
  `/hello`      `helloHandler`   Handles hello requests

`http.HandleFunc("/form", formHandler)` registers the handler with Go's
default HTTP multiplexer (`http.DefaultServeMux`). When a matching
request arrives, the HTTP server invokes the handler. You do not call
`formHandler` directly.

## Handler parameters

A typical handler has this signature:

``` go
func formHandler(w http.ResponseWriter, r *http.Request)
```

-   **`http.ResponseWriter` (`w`)**: Writes the HTTP response back to
    the client.
-   **`*http.Request` (`r`)**: Contains the incoming request's method,
    URL, headers, and body.

Useful request fields include:

-   `r.Method` --- HTTP method, such as `POST`.
-   `r.URL.Path` --- Request path, such as `/form`.
-   `r.RemoteAddr` --- Client network address.

## Parsing form values

`r.ParseForm()` parses URL-encoded form data from the request body and
query parameters.

`r.FormValue("name")` retrieves the value associated with the `name`
key. The key comes from the HTML input's `name` attribute, not its
label.

``` html
<input name="name" type="text">
<input name="address" type="text">
```

Calling `FormValue()` also parses form data when necessary. Calling
`ParseForm()` explicitly is useful when you want to handle parsing
errors.

## Understanding `http.ListenAndServe`

`http.ListenAndServe(":8080", nil)` starts the HTTP server on port
`8080`.

-   `":8080"` specifies the port.
-   `nil` means use `http.DefaultServeMux`, which contains the
    registered routes.
-   The function blocks while the server is running and returns an error
    if it stops.

The syntax:

``` go
if err := http.ListenAndServe(":8080", nil); err != nil {
    log.Fatal(err)
}
```

declares `err` in the `if` statement's initializer. The variable is
scoped to that `if` statement. This is equivalent to declaring `err`
before the `if` and then checking it.

## Static file serving

`http.FileServer(http.Dir("./static"))` creates a handler that serves
files from the `static` directory. Registering it on `/` allows the
browser to request static resources.

For example, opening `http://localhost:8080/` serves the directory's
index page. Submitting the form creates a separate request to `/form`,
which is handled by `formHandler`.

## Key takeaways

-   The browser and Go server communicate through HTTP.
-   Clicking Submit sends a request; it does not directly invoke a Go
    function.
-   `main()` registers routes and starts the server.
-   The HTTP multiplexer routes requests to their matching handlers.
-   `r` contains the request, and `w` is used to write the response.
-   `ParseForm()` parses submitted data; `FormValue()` retrieves fields.
-   `ListenAndServe()` starts the server and blocks while it runs.
-   The file server serves static files; custom handlers process
    application requests.

**Core takeaway:** `main()` starts the server and registers routes. The
browser sends HTTP requests independently. Go's HTTP server routes each
request to the appropriate handler, which processes it and writes a
response.
