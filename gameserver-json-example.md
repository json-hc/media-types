# Example "Controlling Gameservers"

* Author: [DracoBlue](http://dracoblue.net)
* Status: I am working on this. No final version, yet!

This is an example for [json-hc.md](./json-hc.md).

Let's say we own a game hosting service, and want to start and stop game servers. Additionally it should be possible to set server name of the server.

## Client GUI Use-Cases

* Show a list of servers (with pagination)
* Stop or Start servers (Show the button, depending on the state of the server)
* Change the name of a server
 
## Server

### `GET /`

``` json
{
  "http://example.org/rels/servers": "http://example.org/servers"
}
```

### `GET /servers`

Not expanded:

``` json
{
   "next": "http://example.org/servers?page=2",
   "http://example.org/rels/server": [
      "http://example.org/servers/1338"
   ]
}
```

OR expanded:

``` json
{
   "next": "http://example.org/servers?page=2",
   "http://example.org/rels/server": [
      {
        "self": "http://example.org/servers/1338",
        "profile": "http://example.org/rels/server",
        "name": "My Gameserver",
        "is_running": true,
        "http://example.org/rels/stop": "http://example.org/stopped-servers?id=1338"
      }
   ]
}
```

### `GET /servers/1338`

A stopped server:

``` json
{
  "self": "http://example.org/servers/1338",
  "profile": "http://example.org/rels/server",
  "name": "My Gameserver",
  "is_running": false,
  "http://example.org/rels/start": "http://example.org/started-servers?id=1338"
}
```

A started server:

``` json
{
  "self": "http://example.org/servers/1338",
  "profile": "http://example.org/rels/server",
  "name": "My Gameserver",
  "is_running": true,
  "http://example.org/rels/stop": "http://example.org/stopped-servers?id=1338"
}
```

### `PATCH /servers/1338` to set a new name

`Content-Type: application/json-patch+json`
``` json
[
    { "op": "replace", "path": "/name", "value": "My New Gameserver" }
]
``` 

If you expect to PATCH with body `name=My%20New%20Gameserver`, this is *wrong* - see [william durands post on this matter](http://williamdurand.fr/2014/02/14/please-do-not-patch-like-an-idiot/).

Responds with status code 204 NO CONTENT, if it worked.

### `POST /started-servers?id=1338`

Does reply with headers:

```
HTTP/1.1 201 Created
Location: http://example.org/servers/1338/log/1573923
```

in case of success.

Otherwise (with 400 status code):

``` json
{
  "message": "Server is already running!"
}
```

### `GET /servers/1338/log/1573923`

``` json
{
  "self": "http://example.org/servers/1338/log/1573923",
  "profile": "http://example.org/rels/log-entry",
  "message": "Server started",
  "date": "2000-01-01T13:37:55Z"
}
```

## Client (SDK) Use-Cases

### `getServers(): Server[]`

1. GET `/`
2. GET link: `http://example.org/rels/servers`

Return all `http://example.org/rels/server` links as `Server` objects.

### `getServerById(id): Server`

1. GET `/`
2. GET link: `http://example.org/rels/servers` with GET parameter `id=1338` (if id is 1338).

If there is one `http://example.org/rels/server` link, then return it as `Server` object. If it's not available, throw an exception that the server is not found.

### `isServerRunning(id): boolean`

1. GET `/`
2. GET link: `http://example.org/rels/servers` with GET parameter `id=1338` (if id is 1338).
3. GET first link: `http://example.org/rels/server`

If `is_running` property is `true`, return true. Return `false` otherwise. If the server link does not exist, throw an exception.

### `startServer(id): LogEntry`

1. GET `/`
2. GET link: `http://example.org/rels/servers` with GET parameter `id=1338` (if id is 1338).
3. GET first link: `http://example.org/rels/server`
4. POST link: `http://example.org/rels/start`

If there is no `http://example.org/rels/start` link, throw an exception, which states that the server is already started.

The response is a 201 redirect to a log entry and will be returned as LogEntry.

### `stopServer(id): LogEntry`

1. GET `/`
2. GET link: `http://example.org/rels/servers` with GET parameter `id=1338` (if id is 1338).
3. GET first link: `http://example.org/rels/server`
4. POST link: `http://example.org/rels/stop`

If there is no `http://example.org/rels/stop` link, throw an exception, which states that the server is already stopped.

The response is a 201 redirect to a log entry and will be returned as LogEntry.

### `setServerName(server, new_name): Server`

#### self link in the server object

If the `server` object contains a `self` link + etag, make a conditional PATCH request with body:

`Content-Type: application/json-patch+json``
``` json
[
    { "op": "replace", "path": "/name", "value": "My New Gameserver" }
]
``` 

The response has a 204 NO CONTENT status code.

Afterwards retrieve a fresh new copy of server with GET for the `self` link. That's it!

#### no self link in the server object

If the `server` object does not contain a `self` link
retrieve the link for the server:

1. GET `/`
2. GET link: `http://example.org/rels/servers` with GET parameter `id=1338` (if `server.id` is 1338).
3. GET first link: `http://example.org/rels/server`
4. PATCH link: `self` with body:

``` json
[
    { "op": "replace", "path": "/name", "value": "My New Gameserver" }
]
``` 
and header
`Content-Type: application/json-patch+json`


The response has a 204 NO CONTENT status code.

Afterwards retrieve a fresh new copy of server with GET for the `self` link. That's it!

