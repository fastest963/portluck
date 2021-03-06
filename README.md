# Portluck #

[![Build Status](https://travis-ci.org/fastest963/portluck.svg?branch=master)](https://travis-ci.org/fastest963/portluck)

Accepts arbitrary data on a single port via HTTP, HTTPS, WebSockets, or a TCP socket. Optimally data
should be delimited by a `\n` character to speed up connection detection (read below).

Since we're sniffing for the connection type you will need to send at **least** 3 bytes over a raw TCP
socket in order to trigger a `message` event. If those 3 bytes happen to match the beginning of a valid HTTP
method we additionally wait for `\r` or `\n` to determine that it is/isn't an HTTP request.

## portluck.Server ##

### new portluck.Server([messageCallback][, options]) ###
Creates a new server that inherits [https.Server](https://nodejs.org/api/https.html#https_class_https_server).
If `messageCallback` is sent, it will be added as a listener for `"message"` event. See bottom for `options`.

### Event: 'message' ###
Fired when a client sends a message. Event is sent `(message, writer, socket, source)`. `message` is a buffer containing
the message received. `writer` is an instance of `ResponseWriter` and can be used to respond to the message.
`socket` is the source socket that sent the message. You should only read properties off the socket, like
`remoteAddress` and not use the `write`/`end` methods. `source` is either a `net.Socket` or a `http.IncomingMessage`.

### Event: 'clientConnect' ###
Fired when a client connects. Event is sent `(writer, socket, source)`. You should only read properties off the socket,
like `remoteAddress` and not use the `write`/`end` methods. `writer` is an instance of `ResponseWriter` can be
used to respond. The `writer` has `write`, `end`, and `destroy` methods. `source` is either a `net.Socket` or a 
`http.IncomingMessage`.

### Event: 'clientDisconnect' ###
Fired when a client disconnects. Event is sent `socket` which is the socket that disconnected.

### Event: 'clientError' ###
Fired when there's an error emitted from a client. Event is sent `(error, socket)`.

### server.listen(port [,callback]) ###
Accepts the same parameters and options as [http.Server.listen](http://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback).


## ResponseWriter ##

### writer.write(message) ###
Writes `message` to the underlying source socket. `message` can be a string or Buffer. If you plan on
responding to messages asynchronously, you should call `endAfterWrite` since HTTP/HTTPS requests are
automatically closed on the next tick (unless you sent `explicitEnd` option).

### writer.endAfterWrite() ###
If an automatic `end()` is scheduled to happen next tick, calling `doneAfterWrite` prevents that from
happening and waits to automatically call `end()` until the next `write()`. This is not needed if you created
the server with the `explicitEnd` option.

### writer.end([message]) ###
Writes `message`, if passed, to the underlying source socket. If the source was a HTTP/HTTPS request, it is
`end`ed and a response is sent. No more writes are allowed on this writer. If the request was already terminated
on the client side then `message` will be silently dropped and not sent.

### writer.close() ###
Closes (by sending a FIN) the underlying source socket. No more writes are allowed on this writer.

### writer.destroy() ###
Immediately closes the underlying source socket. Similar to `socket.destroy()` No more writes are allowed
on this writer.


## Options ##

### rawFallback ###
Boolean for whether we should fallback to a raw socket connetion if http/tls/websocket isn't detected.

### timeout ###
See [net.Server.timeout](https://nodejs.org/api/http.html#http_server_timeout). Defaults to 2 minutes.
Note: We wait 1 second to try and wait for bytes to determine what type of connection it is. Your `timeout`
applies AFTER that if no data is sent immediately.

### allowOrigin ###
Origin to respond with `Allow-Access-Control-Origin` for. Value can be a string or RegExp. String values can
contain a single `*` for wildcard matching `[a-zA-Z0-9_-]`. **Do not add a protocol (like `https://`).**
Note: `*.example.com` is special and matches `example.com`, `www.example.com`, and `www.staging.example.com`.
Note 2: A origin ending in `:*` will allow any ports. So `example.com:*` would match `example.com` and
`example.com:4000`.

### allowUndefinedOrigin ###
Automatically allow a request if the origin is undefined. You usually want this to be true since browsers only
don't send origin headers for same-domain requests. Defaults to true. If this is false, then the undefined
origin will be passed to `allowOrigin` function/regex.

### explicitEnd ###
If set to `true` you are **required** to call `writer.end()` on every message received. By default, messages are
ended on the nextTick after firing `'message'` event.

### messageLimit ###
Maximum number of bytes per message. This is also the max size of the buffer when sniffing for connection type.
Defaults to 0, which means no limit.

## Todo ##

* Support UDP sockets as well

By [James Hartig](https://github.com/fastest963/)
