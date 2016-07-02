# Golang Neo4J Bolt Driver
[![Build Status](https://travis-ci.org/johnnadratowski/golang-neo4j-bolt-driver.svg?branch=master)](https://travis-ci.org/johnnadratowski/golang-neo4j-bolt-driver) *Tested against Golang 1.4.2 and up*


Implements the Neo4J Bolt Protocol specification:
As of the time of writing this, the current version is v3.1.0-M02

```
go get github.com/johnnadratowski/golang-neo4j-bolt-driver
```
## Usage

*_Please see [the statement tests](./stmt_test.go) or [the conn tests](./conn_test.go) for A LOT of examples of usage_*

### Examples

#### Quick n’ Dirty

```
func quickNDirty() {
	driver := bolt.NewDriver()
	conn, _ := driver.OpenNeo("bolt://localhost:7687")
	defer conn.Close()

	// Start by creating a node
	result, _ := conn.ExecNeo("CREATE (n:NODE {foo: {foo}, bar: {bar}})", map[string]interface{}{"foo": 1, "bar": 2.2})
	numResult, _ := result.RowsAffected()
	fmt.Printf("CREATED ROWS: %d\n", numResult) // CREATED ROWS: 1

	// Lets get the node
	data, rowsMetadata, _, _ := conn.QueryNeoAll("MATCH (n:NODE) RETURN n.foo, n.bar", nil)
	fmt.Printf("COLUMNS: %#v\n", rowsMetadata["fields"].([]interface{}))  // COLUMNS: n.foo,n.bar
	fmt.Printf("FIELDS: %d %f\n", data[0][0].(int64), data[0][1].(float64)) // FIELDS: 1 2.2

	// oh cool, that worked. lets blast this baby and tell it to run a bunch of statements
	// in neo concurrently with a pipeline
	results, _ := conn.ExecPipeline([]string{
		"MATCH (n:NODE) CREATE (n)-[:REL]->(f:FOO)",
		"MATCH (n:NODE) CREATE (n)-[:REL]->(b:BAR)",
		"MATCH (n:NODE) CREATE (n)-[:REL]->(z:BAZ)",
		"MATCH (n:NODE) CREATE (n)-[:REL]->(f:FOO)",
		"MATCH (n:NODE) CREATE (n)-[:REL]->(b:BAR)",
		"MATCH (n:NODE) CREATE (n)-[:REL]->(z:BAZ)",
	}, nil, nil, nil, nil, nil, nil)
	for _, result := range results {
		numResult, _ := result.RowsAffected()
		fmt.Printf("CREATED ROWS: %d\n", numResult) // CREATED ROWS: 2 (per each iteration)
	}

	data, _, _, _ = conn.QueryNeoAll("MATCH (n:NODE)-[:REL]->(m) RETURN m", nil)
	for _, row := range data {
		fmt.Printf("NODE: %#v\n", row[0].(graph.Node)) // Prints all nodes
	}

	result, _ = conn.ExecNeo(`MATCH (n) DETACH DELETE n`, nil)
	numResult, _ = result.RowsAffected()
	fmt.Printf("Rows Deleted: %d", numResult) // Rows Deleted: 13
}
```

## API

*_There is much more detailed information in [the godoc](http://godoc.org/github.com/johnnadratowski/golang-neo4j-bolt-driver)_*

This implementation attempts to follow the best practices as per the Bolt specification, but also implements compatibility with Golang's `sql.driver` interface.

As such, these interfaces closely match the `sql.driver` interfaces, but they also provide Neo4j Bolt specific functionality in addition to the `sql.driver` interface.

It is recommended that you use the Neo4j Bolt-specific interfaces if possible.  The implementation is more efficient and can more closely support the Neo4j Bolt feature set.

The URL format is: `bolt://(user):(password)@(host):(port)`
Schema must be `bolt`. User and password is only necessary if you are authenticating.

You can get logs from the driver by setting the log level using the `log` packages `SetLevel`.


## Pipelining

This interface supports message pipelining, which allows for sending a bunch of requests to neo4j to be processed at once.
You can see examples of this in the tests.

## Dev Quickstart

```
# Put in git hooks
ln -s ../../scripts/pre-commit .git/hooks/pre-commit
ln -s ../../scripts/pre-push .git/hooks/pre-push

# No special build steps necessary
go build

# Testing with log info and a local bolt DB, getting coverage output
BOLT_DRIVER_LOG=info NEO4J_BOLT=bolt://localhost:7687 go test -coverprofile=./tmp/cover.out -coverpkg=./... -v && go tool cover -html=./tmp/cover.out
```

The tests are written in an integration testing style.  Most of them are in the statement tests, but should be made more granular in the future.

In order to get CI, I made a recorder mechanism so you don't need to run neo4j alongside the tests in the CI server.  You run the tests locally against a neo4j instance with the RECORD_OUTPUT=1 environment variable, it generates the recordings in the ./recordings folder.  This is necessary if the tests have changed, or if the internals have significantly changed.  Installing the git hooks will run the tests automatically on push.  If there are updated tests, you will need to re-run the recorder to add them and push them as well.

You need access to a running Neo4J database to develop for this project, so that you can run the tests to generate the recordings.

## TODO

* Connection pooling for driver when not using SQL interface
* Cypher Parser to implement NumInput and pre-flight checking
* More Tests
* Benchmark Tests
* Examples
* Github CI Integration
