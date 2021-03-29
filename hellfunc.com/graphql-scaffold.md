Scaffolding a Go GraphQL gateway

We will go through a very simple Graphql Gateway creation to see how all elements are connected. We'll make all incoming requests in this scaffold respond with some fake data, nothing fancy but enough to let us understand how a GraphQL gateway works. With that in mind we can start creating our project folder.

```
mkdir graphql-scaffold
cd graphql-scaffold
```

Now we initialize our Go project.

```
go mod init graphql-scaffold
```

The `go mod init` command creates a `go.mod` file to track the code's dependencies and make the current directory the root of our project.

## Dependencies

The first dependency of our scaffold will be a GraphQL Go library, this library helps us to construct a GraphLQ server. I've chosen [graphql-go](https://github.com/graph-gophers/graphql-go), it has a minimal API and I found it really easy to use.

```
go get -u github.com/graph-gophers/graphql-go
```

`go get` command is the standard way of downloading and installing packages.

## Schema

In GraphQL, it always starts with a schema. The schema is at the center of every GraphQL server.

```
mkdir schema
touch schema/schema.graphql
```

The schema defines the types of data allowing clients to know which operations can be performed by the server. For our scaffold example, the schema manage an account and we can include the following types:

```
type Account {
  id: ID!
  username: String!
}

input CreateAccountInput {
  username: String!
}

type Query {
  getAccount(id: ID!): Account!
}

type Mutation {
  createAccount(input: CreateAccountInput!): Account!
}

schema {
  query: Query
  mutation: Mutation
}
```

[graphql-go](https://github.com/graph-gophers/graphql-go) library requires the schema as a `string`. We need convert the `schema.graphql` to string. For this end we'll use `ioutil` from Go packages to read the file content.

```
package schema

import (
  "io/ioutil"
  "path/filepath"
)

func String() (string, error) {
  path := filepath.Join("schema", "schema.graphql")
  content, err := ioutil.ReadFile(path)
  if err != nil {
    return "", err
  }
  return string(content), nil
}
```

## Model

A model is a structure that is used to represent data. The model layer can interact with the application's database for example. We can see our models as a bridge connection between the database data and the GraphQL data.

```
// Account represents account data.
type Account struct {
  ID       string
  Username string
}
```

## Resolvers

The GraphQL field's value in our schema needs to be determined by the resolvers in the Go code. A resolver must have one method or field for each field of the GraphQL type it resolves. The method or field name has to be exported and match the schema's field's name in a non-case-sensitive way.

<screenshot folder structure + schema>

```
package resolver

// Resolver is the entry point for all Graphql operations.
type Resolver struct{}

// New creates a new Resolver.
func New() (*Resolver, error) {
	return &Resolver{}, nil
}

// account.go

// AccountResolver resolves Account GraphQL type.
type AccountResolver struct {
	account *model.Account
}

// ID resolves Account id field.
func (r AccountResolver) ID() graphql.ID {
	return graphql.ID(r.account.ID)
}

// Username resolves Account username field.
func (r AccountResolver) Username() string {
	return r.account.Username
}

// mutation.go

type createAccountInput struct {
	Username string
}

type createAccountArgs struct {
	Input createAccountInput
}

// CreateAccount resolves createAccount GraphQL mutation.
func (r Resolver) CreateAccount(ctx context.Context, args createAccountArgs) (*AccountResolver, error) {
	a := &model.Account{
		ID:       "s0m3r4nd0m1d",
		Username: args.Input.Username,
	}
	return &AccountResolver{a}, nil
}

// query.go

type getAccountArgs struct {
	ID graphql.ID
}

// GetAccount resolves getAccount GraphQL query.
func (r Resolver) GetAccount(ctx context.Context, args getAccountArgs) (*AccountResolver, error) {
	a := &model.Account{
		ID:       fmt.Sprint(args.ID),
		Username: "Yo!",
	}
	return &AccountResolver{a}, nil
}
```

## GraphQL handlers

### GraphiQL

Our first GraphQL handler is an in-browser IDE for exploring GraphQL queries. This handler will be really straight forward. We will copy the CDN example from https://github.com/graphql/graphiql/ and write the data as part of the HTTP reply.

```
package gql

import "net/http"

// GraphiQL is an in-browser IDE for exploring GraphiQL APIs.
type GraphiQL struct{}

func (GraphiQL) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write(graphiql)
}

var graphiql = []byte(`
  // https://github.com/graphql/graphiql/blob/main/examples/graphiql-cdn/index.html
`)
```

### GraphQL

This GraphQL struct handles GraphQL API requests over HTTP.

```
package gql

import (
  "encoding/json"
  "net/http"

  "github.com/graph-gophers/graphql-go"
)

// GraphQL struct handles GraphQL API request over HTTP.
type GraphQL struct {
  Schema *graphql.Schema
}

// New creates a new GraphQL handler.
func New(s *graphql.Schema) *GraphQL {
  return &GraphQL{s}
}
```

Before write the ServeHTTP content we need to understand the main elements of a GraphQL query

```
query getAccount($id: ID!) {
  getAccount(
    id: $id
  ) {
    id
    username
  }
}
```

The variables are the part that looks like `$id: ID!` and operation name is the name for our operation, in the previous example `getAccount`.

With this in mind we can create a Go struct to keep the content of a query and the content of our handler.

```
// query represents a single GraphQL query.
type query struct {
  OpName string                 `json:"operationName"`
  Query  string                 `json:"query"`
  Vars   map[string]interface{} `json:"variables"`
}

func (h GraphQL) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  var q query
  if err := json.NewDecoder(r.Body).Decode(&q); err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
    return
  }

  res := h.Schema.Exec(r.Context(), q.Query, q.OpName, q.Vars)
  b, err := json.Marshal(res)
  if err != nil {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
  }

  w.Header().Set("Content-Type", "application/json")
  w.Write(b)
}
```

In the query above we `Decode` the request body into our variable q, then the `Exec` function executes the given query and `json.Marshal` returns the byte JSON encoding of our query response.

## Mux server

The last step is create the Go server and join all the elements. Our Go server will be powered by [gorilla/mux](https://github.com/gorilla/mux).

As we did before with [graphql-go](https://github.com/graph-gophers/graphql-go), we install gorilla/mux using `go get` command.

```
go get -u github.com/gorilla/mux
```

```
package main

func main() {
```

We create the schema as a string using the schema package previously created.

```
sch, err := schema.String()
if err != nil {
  log.Fatal(err)
}
```

Next, we attach the schema with the resolver and we pass it to our GraphLQ handler.

```
h := gql.New(graphql.MustParseSchema(sch, resolver.New()))
```

Now, it's the turn to register the GraphQL URL paths and handlers.

```
r := mux.NewRouter()
r.Handle("/", gql.GraphiQL{})
r.Handle("/graphql", handlers.LoggingHandler(os.Stdout, h))
```

And finally we can define the parameters for running the HTTP server and launch it.

```
srv := &http.Server{
  Addr:           ":4004",
  Handler:        r,
  MaxHeaderBytes: http.DefaultMaxHeaderBytes,
}
log.Fatal(srv.ListenAndServe())
```

We can run the Go server using `go run` command and test the GraphQL scaffold opening the address `http://localhost:4004/`.

```
go run main.go
```

<screenshot of graphiql + result query>
