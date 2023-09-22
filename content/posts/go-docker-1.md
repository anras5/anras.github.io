---
title: "Go with Docker part 1 - Setup"
summary: Building web server in Go and Docker
date: 2023-08-02T21:09:39+02:00
draft: false
---

### What we will build

At the end of this tutorial you will build a simple web server in Go with one endpoint running in a Docker container.

### Prerequisites

This tutorial assumes you have already installed the Go programming language and Docker on your machine. If not, head over to the documentation for steps on how to do that.

- [Go](https://go.dev/dl/)
- [Docker](https://www.docker.com/)

Some basic knowledge of Go and Docker will also come in handy while following the tutorial.

---

# Setting up the project layout

Assuming you have already installed Go on your local machine let's create a directory and run `go mod init` in it.
```bash
mkdir go-with-docker
cd go-with-docker
go mod init github.com/{your-username}/go-with-docker
```
Insert your [github](https://github.com/) username in the right place. Of course, you can also use an IDE like GoLand for this step.

Now you will need to create a structure of directories like this:
```blank
go-with-docker
│   .gitignore
│   go.mod    
│
└───cmd
│   │
│   └───api
│   
└───internal
    |
    └───config
    │
    └───handlers
```
To read more about Go project layout head over to this repository: https://github.com/golang-standards/project-layout

Let's create a `cmd/api/main.go` file and run it to check if everything works:
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello from Go!")
}
```
and run it using this command:
```bash
$ go run cmd/api/main.go
```
You should see this result:
```blank
Hello from Go!
```
in the console.

# Starting web server

All right, now that we have everything working let's start writing some code for the web server. We will start by creating a `internal/config/config.go` file that will contain an `Application` structure which will hold application configuration.
```go
package config

// Application holds the application config
type Application struct {
	Domain string
	DSN    string
}
```

Before defining routes we need to download a 3rd party package - `chi` since we will be using chi router to make our lives easier. Though it is possible to write everything from scratch, developing with chi router is way faster. Let's open command line and run:
```bash
go get -u github.com/go-chi/chi/v5
```
Here is the documention for this package, be sure to check it out: https://github.com/go-chi/chi

You can check in your `go.mod` file that the package was in fact installed. There should be a line looking like this:
```blank
require github.com/go-chi/chi/v5 v5.0.10 // indirect
```

We can now define routes of our application. Right now there will be only one route but in the future we will add more of them in the same file. Make a `cmd/api/routes.go` file and open it.
```go
package main

import (
	"github.com/go-chi/chi/v5"
	"net/http"
)

func routes() http.Handler {
	mux := chi.NewRouter()

	mux.Get("/", func(writer http.ResponseWriter, request *http.Request) {
		_, _ = writer.Write([]byte("Hello from the browser!"))
	})
	
	return mux
}
```
We declare a new function that returns an `http.Handler` that will be used in a second in the `main.go`. In this function we declare an endpoint that will accept GET requests and when called, the provided function will run. In Go, functions that will run when a request is made accept two parameters:
1. writer - we will write a response to the writer
2. request - we can get data from the user
Inside the function we write a string to the writer. At the end of the function we return created router. Let's now use it in the `cmd/api/main.go`:
```go
package main

import (
	"log"
	"net/http"
)

var port = ":8080"

func main() {
	srv := &http.Server{
		Addr:    port,
		Handler: routes(),
	}
	log.Println("Listening on port 8080")
	err := srv.ListenAndServe()
	log.Fatal(err)
}
```

Let's run the whole directory:
```bash
go build ./cmd/api
./api
```

Open your favorite browser and go to `localhost:8080`. You should see `Hello from the browser!` text displayed on the page.

# Handlers

Since in the future we will have a lot of handlers, it would be a wise decision to move them to another package. For this we will create a `internal/handlers/handlers.go` file.
```go
package handlers

import "net/http"

func Home(w http.ResponseWriter, r *http.Request) {
	_, _ = w.Write([]byte("Hello from the browser!"))
}
```

And now we have to remake the `cmd/api/routes.go` file, so it will use `handlers.Home` function.
```go
package main

import (
	"github.com/anras5/go-with-docker/internal/handlers"
	"github.com/go-chi/chi/v5"
	"net/http"
)

func routes() http.Handler {
	mux := chi.NewRouter()

	mux.Get("/", handlers.Home)

	return mux
}
```

# Returning a JSON

Before running everything in a docker container, let's return a json instead of the plain text. We will need to rewrite the `Home` function in the `handlers.go` file.

First of all, let's create a struct with three fields:
```go
payload := struct {
	Status  string `json:"status"`
	Message string `json:"message"`
	Version string `json:"version"`
}{
	Status:  "active",
	Message: "Go API is up and running",
	Version: "1.0.0",
}
```
Each field is annotated with json equivalent so when parsed, the json counterpart will be displayed as the key for a value stored in a particular field. Now we need to parse the payload variable. To achieve this, we will use built-in json package.
```go
out, _ := json.Marshal(payload)
```
For now, we won't be doing anything about the error since we are sure that the parsing of the struct will not fail. At the end we need to set `Content-Type` header to `application/json`. The final `handlers.Home` function will look like this:
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

	// marshal the data
	out, _ := json.Marshal(payload)

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	_, _ = w.Write(out)
}
```
Remember about importing `encoding/json` package in the `handlers.go` file 😊.

We are ready to run our application. After running it the same way as before, go to your browser. You should see something like this:
```blank
{"status":"active","message":"Go Todos API is up and running","version":"1.0.1"}
```

Now it is time to run everything in Docker!

# Dockerfile

In the root of your project directory create a file `Dockerfile`. 
```Dockerfile
FROM golang:alpine

RUN apk update && apk add --no-cache git && apk add --no-cache bash && apk add build-base

WORKDIR /app
COPY go.mod go.sum ./
COPY . .

RUN go mod download
RUN go get github.com/githubnemo/CompileDaemon
RUN go install github.com/githubnemo/CompileDaemon

ENTRYPOINT CompileDaemon -polling -log-prefix=false -build="go build -o apibin ./cmd/api" -command="./apibin" -directory="./"
```

We are using golang alpine as the backbone of our image. Updating apk and installing dependencies will allow us to install CompileDaemon which will scan files in our project directory and compile & run the app when we make changes so we won't have to restart the container every time.

# Docker Compose

In the root of your project directory create a file `docker-compose.yml`.
```yaml
version: '3.9'

services:
  app:
    container_name: golang_container
    build: .
    ports:
      - '8080:8080'
    restart: on-failure
    volumes:
      - .:/app
```
We bind port 8080 on the host with 8080 in the container and create a volume so changes made on the host will be seen in the container.

Start the docker by running
```bash
docker-compose up --build
```
in the command line.

After the build stage you should see:
```blank
Attaching to golang_container
golang_container  | Running build command!
golang_container  | Build ok.
golang_container  | Restarting the given command.
golang_container  | 2023/08/02 00:00:00 Listening on port 8080
```

Since the port 8080 is bound, you can open a browser one more time and head over to the `localhost:8080` to see that the API is successfully running in the container! 

You can find full code for this tutorial in my github: https://github.com/anras5/go-with-docker

Click here to see the next tutorial: [Part 2](/posts/go-docker-2/)