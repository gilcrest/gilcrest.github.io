---
title: "Basic Redis Examples with Go"
date: 2018-07-17T20:38:00-04:00
description: Basic Redis Examples with Go
menu:
  sidebar:
    name: Basic Redis Examples with Go
    identifier: basic-redis-examples-go
    parent: archive
    weight: 200
---


Redis is pretty great. It is the #1 most loved database for the second year in a row, [according to a recent Stack Overflow survey](https://insights.stackoverflow.com/survey/2018/#technology-most-loved-dreaded-and-wanted-databases). I decided it was high time I taught myself how to use it with Go.

There are a number of libraries in the Go ecosystem for Redis, but the two most popular are [go-redis](https://github.com/go-redis/redis) and [redigo](https://github.com/gomodule/redigo). Each library has a decent amount of stars, contributors, etc. but from what I can tell redigo seems to have a slight edge in terms of documentation and community acceptance. For instance, the mighty Redis Labs, has a couple of posts describing interacting with Redis via Go using *redigo* — it’s not a direct endorsement, but pretty close… I tried both libraries actually, but found redigo was better documented. That said, in order to get up and running quickly with redigo, having to troll through godoc was a bit cumbersome (I’m still getting my head around analyzing godoc TBH). In this post, I’ll attempt to give some simple examples for *redigo* that may prove helpful to some.

Below I take you through creating a connection pool, testing connectivity via the `PING` command, adding a simple Key:Value pair via the `SET` command, retrieving a given value given the `GET` command and finally storing a struct as JSON using the `SET` command and retrieving the same struct with the `GET` command.

TL;DR — For the full main.go with all the examples below, you can find them [here](https://github.com/gilcrest/redigo-example/blob/master/main.go).

## POOL

To establish connectivity in redigo, you need to create a `redis.Pool` object which is a pool of connections to Redis. In order to do this, you can use something like the following:

```go
func newPool() *redis.Pool {
    return &redis.Pool{
        // Maximum number of idle connections in the pool.
        MaxIdle: 80,
        // max number of connections
        MaxActive: 12000,
        // Dial is an application supplied function for creating and
        // configuring a connection.
        Dial: func() (redis.Conn, error) {
            c, err := redis.Dial("tcp", ":6379")
            if err != nil {
                panic(err.Error())
            }
            return c, err
        },
    }
}
```

Use the Get method of the Pool object to grab a connection from the pool. Per the redigo documentation, “The application must call the connection Close method when the application is done with the connection.”

```go
func main() {
    pool := newPool()
    conn := pool.Get()
    defer conn.Close()
    err := ping(conn)
    if err != nil {
        fmt.Println(err)
    }
...
```

## PING

If you wish you to simply check connectivity, you can use Redis’ `PING` command.

```go
// ping tests connectivity for redis (PONG should be returned)
func ping(c redis.Conn) error {
    // Send PING command to Redis
    pong, err := c.Do("PING")
    if err != nil {
        return err
    }

    // PING command returns a Redis "Simple String"
    // Use redis.String to convert the interface type to string
    s, err := redis.String(pong, err)
    if err != nil {
        return err
    }

    fmt.Printf("PING Response = %s\n", s)
    // Output: PONG

    return nil
}
```

To break down what’s happening here — the function takes in a redis Conn type (connection). We are calling the Do method of Conn, which takes a Redis command as it’s first argument. The redigo client communicates to Redis using Redis’ [RESP](https://redis.io/topics/protocol#resp-protocol-description) protocol. Redis then responds with one of several types (Simple Strings, Errors, Integers, Bulk Strings and Arrays), which are mapped to Go types. The response from the Do method does not give back the specific Go type though, but rather an interface{} type. We can use a type assertion to determine the response type from the Do method, but seeing as we know from the [Redis documentation](https://redis.io/commands/ping#return-value) that the Return Value from PING is a simple string, we can use one of redigo’s handy helper functions (`redis.String`) to perform a type conversion to string for us.

For illustrative purposes in my example above, I executed the helper `redis.String` function separately from the `Do` method as I found it easier to understand on my first pass. In reality, you’ll use the redigo provided helper functions to wrap your `Do` method calls (provided you know the response type), as I have done in the example below. Any remaining examples will use this shortened form where appropriate.

```go
// ping tests connectivity for redis (PONG should be returned)
func ping(c redis.Conn) error {
    // Send PING command to Redis
    // PING command returns a Redis "Simple String"
    // Use redis.String to convert the interface type to string
    s, err := redis.String(c.Do("PING"))
    if err != nil {
        return err
    }

    fmt.Printf("PING Response = %s\n", s)
    // Output: PONG

    return nil
}
```

## SET

Use the Redis `SET` command to add a key:value pair to Redis. Below is a trivial example of adding a key (“Favorite Movie”) and a string value for it (“Repo Man”) as well as an int value (1984 as the movie Release Year). The blank identifier is used for the reply as we only need to check for errors (“OK” is the only thing that comes back on a successful `SET` command reply).

```go
// set executes the redis SET command
func set(c redis.Conn) error {
    _, err := c.Do("SET", "Favorite Movie", "Repo Man")
    if err != nil {
        return err
    }
    _, err = c.Do("SET", "Release Year", 1984)
    if err != nil {
        return err
    }    
    return nil
}
```

## GET

In order to retrieve a value for a given key, use the Redis `GET` command. Some simple examples are below, including an example where no results are retrieved. We can check for `redis.ErrNil` to determine if nothing is returned and should handle appropriately.

```go
// get executes the redis GET command
func get(c redis.Conn) error {

    // Simple GET example with String helper
    key := "Favorite Movie"
    s, err := redis.String(c.Do("GET", key))
    if err != nil {
        return (err)
    }
    fmt.Printf("%s = %s\n", key, s)

    // Simple GET example with Int helper
    key = "Release Year"
    i, err := redis.Int(c.Do("GET", key))
    if err != nil {
        return (err)
    }
    fmt.Printf("%s = %d\n", key, i)

    // Example where GET returns no results
    key = "Nonexistent Key"
    s, err = redis.String(c.Do("GET", key))
    if err == redis.ErrNil {
        fmt.Printf("%s does not exist\n", key)
    } else if err != nil {
        return err
    } else {
        fmt.Printf("%s = %s\n", key, s)
    }

    return nil
}
```

## STRUCT (SET)

For my purposes of building a look-aside cache, I want to store objects in cache in their entirety in Redis. There seem to be a few different viewpoints on doing this — one where you store your object using a Redis Hash data type and the `HMSET` command. This is nice because, if need be you can update individual values within the object independently. I have seen examples of people using the redigo `AddFlat` method of the `redigo.Args` type to accomplish this, but also noted in redigo’s [FAQ](https://github.com/gomodule/redigo/wiki/FAQ#does-redigo-provide-a-way-to-serialize-structs-to-redis) that redigo does not actually provide a way to serialize structs to Redis, so I stayed away from trying this.

For my purposes, I am ok with just storing the data as an object in its entirety and thus decided to store my data as JSON using the `SET` command.

The example below shows storing a user, with username “otto”, with a key of “user:otto”. Use `json.Marshal` to serialize your object to JSON and store the serialized version as the value.

```go
func setStruct(c redis.Conn) error {

    const objectPrefix string = "user:"

    usr := User{
        Username:  "otto",
        MobileID:  1234567890,
        Email:     "ottoM@repoman.com",
        FirstName: "Otto",
        LastName:  "Maddox",
    }

    // serialize User object to JSON
    json, err := json.Marshal(usr)
    if err != nil {
        return err
    }

    // SET object
    _, err = c.Do("SET", objectPrefix+usr.Username, json)
    if err != nil {
        return err
    }

    return nil
}
```

## STRUCT (GET)

The example below shows retrieving the object using the `GET` command and then deserializing it with `json.Unmarshal`

```go
func getStruct(c redis.Conn) error {

    const objectPrefix string = "user:"

    username := "otto"
    s, err := redis.String(c.Do("GET", objectPrefix+username))
    if err == redis.ErrNil {
        fmt.Println("User does not exist")
    } else if err != nil {
        return err
    }

    usr := User{}
    err = json.Unmarshal([]byte(s), &usr)

    fmt.Printf("%+v\n", usr)

    return nil

}
```

```json
{Username:otto MobileID:1234567890 Email:ottoM@repoman.com FirstName:Otto LastName:Maddox}
```

---

That’s it for now. Keep in mind, I wrote this main and these little functions in a somewhat non-idiomatic way just to aid in the examples, you should streamline this in a real app.

I also created a similar `main.go` for the go-redis client, that you can find [here](https://github.com/gilcrest/go-redis-example/blob/master/main.go). For `go-redis`, storing structs as Redis hashes is easier, as you can pass a map of strings type to the `HMSET` command.
