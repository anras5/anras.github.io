---
title: "Go with Docker part 2 - JSON"
summary: Adding json module
date: 2023-09-19T17:48:37+02:00
draft: false
---

### What we will build

At the end of this tutorial, your web app will be equipped with a JSON module, simplifying the process of parsing payloads.

### Prerequisites

This tutorial assumes you have followed [Go with Docker part 1](/posts/go-docker-1)

---

# Creating json.go file

This file will help us serialize and deserialize data into structs with ease. Open `internal/config` package and in this directory create a new file `json.go`

## Writing JSON

To begin, we will create a function that allows us to write data in the form of JSON to the ResponseWriter. We start by declaring the WriteJSON function with a capital first letter so that the function is visible outside the config package.

```go
// WriteJSON is used to write json data to the ResponseWrite
func WriteJSON(w http.ResponseWriter, status int, data interface{}, customHeaders ...http.Header) error {
	return nil
}
```
This function takes four parameters:

1. The ResponseWriter that will be used to write the response.
2. The HTTP status code that will indicate the success of the response.
3. An interface to hold any data that needs to be serialized.
4. Custom HTTP headers (zero or more values).

**Now, let's define the body of the function:** \
We need to marshal the data into a JSON response.
```go
// marshal data
out, err := json.Marshal(data)
if err != nil {
    return err
}
```

Next, we should append custom headers to the Response Writer headers:

```go
// append custom headers
if len(customHeaders) > 0 {
    for _, header := range customHeaders {
        for key, values := range header {
            for _, value := range values {
                w.Header().Add(key, value)
            }
        }
    }
}
```

Finally, we set the 'Content-Type' header to 'application/json' and write the JSON data:

```go
// write to the Response Writer
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(status)
_, err = w.Write(out)
if err != nil {
    return err
}
```

This code should work effectively for writing JSON data to the ResponseWriter with the specified status code and custom headers.


## Reading JSON

We will create a function `ReadJSON` that allows us to read data in the form of JSON from a http.Request to an interface.
```go
func ReadJSON(w http.ResponseWriter, r *http.Request, data interface{}) error {
	maxBytes := 1024 * 1024
	r.Body = http.MaxBytesReader(w, r.Body, int64(maxBytes))

	decoder := json.NewDecoder(r.Body)
	decoder.DisallowUnknownFields()

	err := decoder.Decode(data)
	if err != nil {
		return err
	}

	err = decoder.Decode(&struct{}{})
	if err != io.EOF {
		return errors.New("body must contain only a single JSON value")
	}

	return nil
}
```

In this function we decode JSON data from a http.Request and ensure that the requwest body conforms to specific constraints. In the first step we limit request body size to maximum of 1MB. This is made to prevent potential DoS attacks. Next step is decoding the body of the request and disallowing unknown fields to enforce data structure conformity. We also check if the body has only one JSON value.

## Error JSON

Another helpful function is `ErrorJSON` that will allow us to quickly send errors to the users. First we need to create

```go
func ErrorJSON(w http.ResponseWriter, err error, status ...int) error {
	statusCode := http.StatusBadRequest
	if len(status) > 0 {
		statusCode = status[0]
	}
	payload := struct {
		Error bool `json:"error"`
		Message string `json:"message"`
	}{
		Error: true,
		Message: err.Error(),
	}

	return app.WriteJSON(w, statusCode, payload)
}
```

This function is responsible for generating and sending JSON error responses to clients. We can customize status code and send error in the "message" attribute.

# Using `json.go` in `handlers.go`

Now, we can update the `Home` function in handlers.go by using the previously defined functions, making it more concise. New `Home` function can look like this:
```go
func Home(w http.ResponseWriter, r *http.Request) {
	payload := struct {
		Status  string `json:"status"`
		Message string `json:"message"`
		Version string `json:"version"`
	}{
		Status:  "active",
		Message: "Go Todos API is up and running",
		Version: "1.0.1",
	}

	_ = config.WriteJSON(w, http.StatusOK, payload)

}
```

You can find full code for this tutorial on my github: https://github.com/anras5/go-with-docker