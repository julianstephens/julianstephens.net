---
author: Julian Stephens
pubDatetime: 2022-09-15T22:00:52.737Z
title: Building an E-Commerce Web App with Golang, React, Docker, and Stripe - Pt 1
postSlug: building-ecommerce-web-app-pt-1
tags:
  - programming
  - fullstack
  - javascript
  - node 
  - golang
  - react
  - docker
  - tutorial
description: A beginner-friendly tutorial using React, Golang, Docker, and Stripe
---

## Part One: Planning and Setup
### Intro
In this tutorial, we’ll be creating an online clothing store in brutalist style using React, Golang, Docker, and Stripe. If you’ve never used any of these tools before that’s completely okay! This is written with beginners in mind, and all of the code needed to complete this project is provided along with step-by-step instructions on setting it up. Explanations are provided for most steps. However, for detailed information on the tools themselves and their usage, please see the associated documentation.

This is a big project, so the tutorial is split into several parts. In this first module, we’ll get the environment setup and basic project infrastructure laid before jumping into the heavier programming. Code snippets are provided inline with the explanation, but the source code for the finished project is also provided [here](https://github.com/julianstephens/stripe-shopping-app). If you have any questions or problems following this tutorial, please open an issue on the project GitHub repo. 

Let’s get started!

### API & DB Planning
I’ve been burned by jumping into a new project without a plan enough times to get a little sweaty without having a rough idea of where I’m going. Before we start writing any code, let’s get basic sketch of what features we’d like to have and what our backend will look like. 

**Features:**
- As a user, I can create an account.
- As a user, I can login to my existing account
- As a user, I can view a list of products available for purchase.
- As a user, I can add items to my cart.
- As a user, I can view my cart.
- As a user, I can complete a purchase.
- As a user, I can like a product.
- As a user, I can view a list of my liked products.

That seems like a good start to get the basic structure of an e-commerce store in place. From here, we can map out what API routes we’ll need for each item.

| Action     | Route     | HTTP Method     |
| ---------- | ---------- | ---------- |
| Create account       | /auth/register       | POST       |
| Login       | /auth/login       | POST       |
| View products       | /products       | GET       |
| View cart       | /user/<user_id>/cart       | GET       |
| Create a cart       | /user/<user_id>/cart       | POST       |
| Add to cart       | /user/<user_id>/cart       | PUT       |
| Like product       | /user/<user_id>/likes/<product_id>       | POST       |
| View likes       | /user/<user_id>/likes       | GET       |
| Complete purchase       | /user/<user_id>/create-session       | POST       |

Great, that’s a rough sketch of our API. We still have some more planning to do though. We need to figure out how to store all the data our app is generating. Our next task is to mockup our database. To keep things simple, a user can only have one cart and one list of likes at a time. 

```dbml
Table users {
	id serial [pk]
	email text [unique]
	password text [not null]
}

Table carts {
	id serial [pk]
	user_id integer [ref: > users.id]
}

Table products {
	id serial [pk]
	stripe_prod_id text [unique]
	description text
	price numeric(28, 10) [not null]
}

Table images {
	id serial [pk]
	url string
}

Ref cart_items: carts.id <> products.id // many-to-many relationship betweeen carts & products
Ref user_likes: users.id <> products.id
Ref product_images: product.id < images.id // one-to-many relationship between products & images
```
**Explanation:**

1. `Project stripe_shopping_app {}`: this block defines our database name and type 
2. `Table users{}`: to keep it simple the only user data we’ll store is their login credentials
3. `Table carts{}`: this table will act as a permanent store for cart objects. we’ll also store them in redis for short-term access
4. `Table products{}`: the products table will house much of the same information as our Stripe site but will let us access the info without making potentially costly API calls
5. `Table images{}`: the images table extracts product image URLs so that our product app object will hold a list of images
6. `Ref carts_products`: this creates a join table called ‘carts_products’
7. `Ref users_likes`: 'user_likes' also becomes a join table
8. `Ref product_images`: 'product_images' also becomes a join table

<br />

That’s all the planning we’ll do for now. Time for the fun part!

### Project Setup
We'll be using Go (aka Golang) for the backend and React on the frontend. If you're not familiar with Go, there's an awesome introduction to the language called [A Tour of Go](https://tour.golang.org/welcome/1) on the site.

To install Go, follow the [installation instructions](https://go.dev/doc/install) for your operating system. Once you've got it properly installed on your machine, we'll create a project directory for our app. If you plan to put your work on GitHub, then create `$GOPATH/src/github.com/your_github_username/stripe-shopping-app`, otherwise `$GOPATH/src/stripe-shopping-app`.

> Note: Throughout this tutorial shell code blocks will always start in the project root. Non-shell blocks will contain a comment on the first line with the file’s location relative to the project root.

```shell
$ cd
$ mkdir -p $GOPATH/src/github.com/your_github_username/stripe-shopping-app
$ cd $GOPATH/src/github.com/your_github_username/stripe-shopping-app
```

Next, we'll create the `server/` directory, which is where our Go backend will live. To get started, the first thing we need to do is initialize the project. If the following lines are confusing, check out [this article](https://blog.golang.org/using-go-modules) on Go modules.

```shell
$ cd server
$ go mod init github.com/your_username/stripe-shopping-app/server
$ cd ..
```

Once you've initialized your project, you can create a file called `main.go`. This is where we'll write the Go code for our project.

Now that we've go our Go project setup, let's get our React app up and running. If you don't already have [Node.js](https://nodejs.org/en/download/),
[npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm), and [npx](https://www.npmjs.com/package/npx) on your machine, you'll need to download and install them before proceeding to the next step.

```shell
$ npx create-react-app client
```

![npx-create-react-app](//images.ctfassets.net/skkcb7wqibsg/3Z2gkwLTojU2eqYe7YeRpu/0192ed6c19455440028f00076ecf6e79/npx_create_react_app.jpg)

Awesome, you're ready to go! Here's what our project structure looks like now.

```
stripe-shopping-app
|	README.md
|
|___client/
|	|_  node_modules/
|	|_  public/
|	|_  src/
|	|_  .gitignore
|	|_  package.json
|	|_  package-lock.json
|	|_  README.md
|
|___server/
|	|_  go.mod
|	|_  go.sum
```

### Create Go server
The first thing we’ll do is write the basic structure of our Go server. This will let us test out our Go setup and make sure everything is configured properly.
#### Install Base Dependencies
Let’s go ahead and install a few of the packages that we know we’ll need soon. 

| Package    | Usage                              | Notes|
| ---------- | --------------------|------------------------- |
| stripe-go  | Stripe client for payment          | Be sure to go to the [Stripe docs](https://stripe.com/docs/api) and grab the current version for the Go client (for me its 73).                                                   |
| godotenv   | Use .env for environment variables |                                                                                                                                                                                   |
| echo       | Web framework                      | I recommend scanning through the [docs](https://echo.labstack.com/guide/) if you’re not already familiar with Echo. You should also double check the version number here as well. |
| gorm       | Database ORM                       | Checkout the [docs](https://gorm.io/docs/)                                                                                                                                        |
| validator  | Struct type validation             |                                                                                                                                                                                   |
| golang-jwt | JWT utilities                      |                                                                                                                                                                                   |

```shell
$ cd server
$ go get github.com/stripe/stripe-go/v<latest-version-number>
$ go get github.com/joho/godotenv
$ go get github.com/labstack/echo/v4
$ go get gorm.io/gorm
$ go get gorm.io/driver/postgres
$ go get gopkg.in/validator.v2
$ go get github.com/golang-jwt/jwt
```
#### Create `.env` 
We also need to setup the environment variables for our Go project. You can use [this site](https://www.grc.com/passwords.htm) to generate a JWT secret key. 

> Note:  Make sure to add `.env` to your `.gitignore` if you plan to upload your code to GitHub or other repository host.

In order to use the Stripe client, we need to provide the API credentials attached to our seller accounts. If you don't already have an account, you can create one [here](https://dashboard.stripe.com/register). Once you've created an account or logged in, navigate to 'Developers' -> 'API Keys'. You should see a publishable and secret key for your test account; copy them to your environment variables. 

![stripe-dashboard](//images.ctfassets.net/skkcb7wqibsg/6q0aa5XRBgl6HseUHs5IU7/3fe280dbf8bb1cc669e56f90c68b84ef/stripe_dashboard.jpg)

```
# server/.env

STRIPE_PUBLISHABLE_KEY=your_stripe_publish_key
STRIPE_SECRET_KEY=your_stripe_secret_key

API_PORT=8001

DB_HOST=db
DB_PORT=5432
DB_NAME=stripe_shopping_app
DB_USER=postgres
DB_PASSWORD='your_db_password'

CLIENT_DOMAIN=http://client:3000

JWT_SECRET_KEY=your_jwt_key

```

Great, we have all of our dependencies installed and our environment variables setup; let's start coding.
#### Create `/ping`
Before we get too far, let's do a quick sanity check to make sure everything is setup correctly. We'll create a route `/ping` to test calling our API. To keep our code easy to read, we’re going to extract our router setup to its own package and then reference it in `main`.

Our router file will have a single exported function `Init()` that creates our API definition. We’re going to put all our routes in a group called `api`. For now we just have the ping route. 

> Note: In Go, functions names should be camel-cased. If the first word is capitalized, then the function is exported and available to use outside of the package.

```go
// server/router/router.go

package router

import (
	"net/http"

	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
)

func Init() *echo.Echo {
	e := echo.New()

	e.Use(middleware.Logger())
	e.Use(middleware.CORSWithConfig(middleware.DefaultCORSConfig))

	api := e.Group("/api")
	{
		api.GET("/ping", func(c echo.Context) error {
			return c.JSON(http.StatusOK, &utils.HttpResp{Message: "Ping received", Data: nil})
		})
	}

	return e
}
```

Each route method takes a handler function that processes and responds to an HTTP request sent to that address. For this project our HTTP responses will always have two parts, a message string and any applicable data. In the JSON response for our `/ping` route, we’re using a struct that defines that common response pattern. 
#### Add `utils` package
We’ll create a package called `utils` to hold models and functions that are shared across the application. Here we can put our response struct and any other reusable functions or structs we need.

```go
// server/utils/utils.go

package utils

import (
	"log"
	"os"

	"github.com/joho/godotenv"
)

func GetEnvVar(key string) string [](){
	if err := godotenv.Load("./server/.env"); err != nil {
		log.Fatal("error loading .env file")
	}
	return os.Getenv(key)
}

func PanicError(err error) {
	if err != nil {
		panic(err)
	}
}

type HttpResp struct {
	Message string      `json:"message"`
	Data    interface{} `json:"data"`
}
```

In addition to our HttpResp struct, we also added two helper functions that we’ll use throughout the project. `GetENvVar()` loads our `.env` and provides the associated value for a given key. `PanicError()` is simply a wrapper around a common error handling pattern. 
#### Start server
Now that we’ve gotten our router setup, we can create and start the server in `main.go`.

```go
// server/main.go

package main

import (
	"github.com/<your_gh_username>/stripe-shopping-app/server/router"
	"github.com/<your_gh_username>/stripe-shopping-app/server/utils"
)

func main() {
	// Load server port
	PORT := utils.GetEnvVar("API_PORT")

    // Initialize echo
    e := router.Init()

    // Start the server
    e.Logger.Fatal(e.Start(":" + PORT))
}
```

**Explanation:**

1. Every Go source file starts with a package declaration  `package package_name`. In this case, we're declaring this file as part of the `main` package.
2. Our `main()` function is the entry point for our Go app. It initializes our router and starts the server on the port defined in `server/.env`.

To run the project we'll use the command `go run main.go`.
```shell
$ cd server
$ go run main.go
```

![start-server](//images.ctfassets.net/skkcb7wqibsg/6rB5MGFRieItx26jGAJIoN/1671b54f69016092587598b1ad13ed76/start_server.jpg)

#### Test `/ping`
To query our API, I’m going to use [Postman](https://www.postman.com/downloads/), but you can also use cURL in the terminal, your browser, or any other tool. For convenience, you can download and import the Postman collection for this project [here](https://github.com/julianstephens/stripe-shopping-app/blob/378da90de82d3b77839f4476080de94996ba6f12/stripe_shopping_app.postman_collection.json). Our API is available at `http://localhost:8001/api/`.

Let’s try making a GET request to our ping route (`http://localhost:8001/api/ping`). You should get a JSON response with status code 200 and the message “Received”.  

![ping](//images.ctfassets.net/skkcb7wqibsg/5M24aIqLdKVxyO9XsKeNAI/4cea20445f735b3093a93daf0959129e/ping.jpg)

\* *I created an environment variable called base and set it to `http://localhost:PORT/api` for simplicity.*

### Dockerize Dev Environment
We’re going to use Docker to help us standardize the build processes and dependency management across development and production environments. To do this we’re going to create three containers: client (React frontend), server (Go REST API), and db (Postgres database). Once we have our Dockerfiles written, we can use [Compose](https://docs.docker.com/compose/) to mange and run the containers. Make sure you have Docker installed and have Compose V2 enabled before moving forward in the tutorial.

For now, we’re only going to be working with server and db, so we’ll wait to add the configuration for the client container. To get started on the other two, let’s create a folder for our Dockerfiles.

```shell
$ mkdir .docker
$ cd .docker
$ touch server.dockerfile
```
#### Server Container
In our Dockerfile, we're essentially going to describe the process of building and running our Go app. We’re also going to use a tool called [air](https://github.com/cosmtrek/air) to give us live reloading so we don’t have to restart the container every time we make changes to our server.

```dockerfile
# .docker/server.dockerfile

FROM golang:1.18-alpine as base

# Load build arg
ARG APP_HOME

# Create development stage and set app location in container to /app 
FROM base as development
WORKDIR $APP_HOME

# Add bash shell and gcc toolkit
RUN apk add --no-cache --upgrade bash build-base

# Install air
RUN go install github.com/cosmtrek/air@latest

# Copy go.mod and install go dependencies
COPY go.mod ./
RUN go mod download

# Copy source files
COPY . .

# Build and run binary with live reloading
CMD ["air"]
```

To setup air,  create a file called `.air.toml` in your project root. Here, we’ll define our project’s run configuration. See the air [docs](https://github.com/cosmtrek/air/blob/master/air_example.toml) for explanation.

```toml
# .air.toml

# Working directory
# . or absolute path, please note that the directories following must be under root.
root = "./server"
tmp_dir = "tmp"

[build]
# Just plain old shell command. You could use `make` as well.
cmd = "cd server && go build -o ./tmp/main ."
# Binary file yields from `cmd`.
bin = "./server/tmp/main"
# Don't save right away (ms)
delay = 1000
# Watch these filename extensions.
include_ext = ["go"]
# Ignore these filename extensions or directories.
exclude_dir = ["tmp"]
# Exclude specific regular expressions.
exclude_regex = ["_test\\.go"]
# Save logs to file
log="air_errors.log"

[misc]
# Delete tmp directory on exit
clean_on_exit = true
```
#### Docker Compose
To actually manage and run our containers, we’re going to use Docker Compose; this requires a configuration file called `docker-compose.yml` located at the project root.

```yaml
# docker-compose.yml

services:
  db:
    image: postgres:14-alpine
    container_name: db
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - stripe_pgdata:/var/lib/postgresql/data
    ports: 
      - ${DB_PORT_FORWARD}:${DB_PORT}
    restart: always

  server:
    container_name: server
    hostname: api
    build:
      context: ./server
      dockerfile: ../.docker/server.dockerfile
      target: ${APP_ENV}
      args:
        APP_HOME: ${APP_HOME}
    ports:
      - ${API_PORT_FORWARD}:${API_PORT}
      - ${API_DEBUG_PORT_FORWARD}:${API_DEBUG_PORT}
    volumes:
      - .:${APP_HOME}
    links:
      - db

volumes:
  stripe_pgdata:
    external: true
```
**Explanation:**

1. `db`: The first service (aka container) is called `db` and it’s based on the `postgres:14-alpine` image. Notice that we’ve mounted a volume called `stripe_pgdata` to the container in order to persist database data after container shutdown.
2. `server`: This container is built from the definition we provided in `.docker/server.dockerfile`. We also opened a public port ${API_PORT} that our API will run on and forward that port to ${API_PORT_FORWARD} to avoid conflicts. From inside a container, we’ll be able to reach this service at `http://server:APP_PORT`; outside the container (e.g. Postman or browser) it can be reached at `http://localhost:APP_PORT_FORWARD`.  We also mounted the project directory to a volume mapped to ${APP_HOME}. This will add our project files to the container and sync changes between our local copy and the files inside the container. 
3. `volumes`: The volumes section tells Compose that the ‘stripe_pgdata’ volume we attached to `db` was created externally and shouldn’t be recreated when Compose builds or tears down containers.  

Before we can build our images and start the containers, we need to define the environment variables and create the volume for the database.  To create the volume:

```shell
$ docker volume create stripe_pgdata
```

For our environment variables, we’ll create a `.env` file at our project root. 

```env
# .env

APP_HOME=/go/src/github.com/<your_github_username>/stripe-shopping-app
APP_ENV=development

DB_NAME=stripe_shopping_app
DB_PASSWORD='<your_db_password>'
DB_PORT_FORWARD=3502
DB_PORT=5432

API_PORT_FORWARD=3500
API_PORT=8001

API_DEBUG_PORT_FORWARD=2345
API_DEBUG_PORT=2345
```
#### Run Dev Environment
[VSCode](https://code.visualstudio.com/) and [Goland](https://www.jetbrains.com/go/) both have GUI tools for interacting with Docker that are extremely useful. However, for generalization, I’m going to use the CLI.  

```shell
$ docker compose up -d --build
```

This will build the Docker images and start the containers in detached mode (i.e. in the background). If you open up the Docker Desktop app, you should see your containers running. Clicking on the container name will allow you to view the service logs, configuration, and stats. Now, if you go back to Postman and run the ping request again you should still receive the 200 response with the message “Received”.  **Be sure to update the port to the forwarded port before running any requests.** 

![docker-up-server-db](//images.ctfassets.net/skkcb7wqibsg/6axlnX1yPjPc1eFbyPTUpg/d202081450f7e310a5a5aca03ff37cdb/docker_up_server_db.jpg)

—

That concludes part one of the tutorial! By this point you should: 

1. Have a functioning Go API with an active route at `/api/ping`. 
2. Setup a react project in `client/` using create-react-app. 
3. Dockerized your Go server, React frontend, and Postgres database.

<br />

In the next part of the tutorial, we’ll finish the backend implementation for our web app.
