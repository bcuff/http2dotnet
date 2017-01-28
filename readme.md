http2dotnet
===========

This library implements the HTTP/2 and HPACK protocol for .NET standard.
The goal of the library is to cover all protocol handling parts of the HTTP/2
([RFC7540](http://httpwg.org/specs/rfc7540.html)) and HPACK
([RFC7541](http://httpwg.org/specs/rfc7541.html)) specifications.
The goal of this library is NOT to provide a ready-to-use HTTP/2 server or
client framework with concrete Request/Response abstractions. Instead it should
enable other .NET libraries and applications to easily integrate HTTP/2 support
by encapsulating the protocol support in easy to use and flexible .NET classes.
Examples for building simple applications on top of the library are provided
within the repository.

Current state
-------------
This library is currently in an experimental state. The majority of HTTP/2
features have been implemented and there is already a pretty good test coverage.
It was tested in a h2c with prior knowledge configuration against **nghttp2**
and **h2load**. It was however not yet tested with an encrypted connection and
a browser based client due a missing SSL connection with ALPN negotation.
Various features (especially for client side usage) have not yet been
implemented (see Current limitations).

Design goals
------------

- Enable an easy integration of HTTP/2  into different frameworks and
  applications (ASP.NET core, other webservers, gRPC, etc.)
- Provide an abstract interface for HTTP/2 Streams, on which custom Request
  and Response classes can be built. The Stream abstraction supports all
  features of the HTTP/2 specification, including sending HEADERS in both
  directions, flow-controlled DATA and trailing HEADERS. The Request and
  Response are handled fully independet, which enables features like pure server
  side data streaming.
- IO layer independence:  
  This library uses an abstract IO interface, which is defined in the
  `ByteStreams.cs` source file.  
  The IO layer can be implemented by .NET System.IO.Streams, Pipelines,
  TCP sockets, TLS sockets and other IO sources.  
  The abstract IO interface is inspired by the Go IO interfaces. It was chosen
  for simplicity, high performance (async reads/writes can be avoided) and 
  because it can easily provide the necessary flow control behavior.
- Correctness and reliability:  
  All HTTP/2 frames are validated according to the HTTP/2 specification.  
  Full backpressure behavior is provided for HTTP/2 DATA
  frames (through the specified flow control mechanism) as well as for other
  HTTP/2 frames (by suspending the reception of further frames until a suitable
  response for a frame could be sent).
- High performance:  
  To achieve a high performance and low overhead the library tries to minimize
  dynamic allocations whereever possible, e.g. through the use of `ValueTask`
  structures and reusing of buffers and data structures. If possible the library
  also avoids async operations by trying nonblocking operations first to avoid
  scheduling and continuation overhead.
- Full support of all HTTP/2 features (however not yet everything is implemented)

Usage
-----

The library exposes 2 important classes:
- `Connection` represents a HTTP/2 connection between a client and the server.  
  HTTP/2 connections are created on top of bidirectional streams. For encrypted
  HTTP/2 (the only variant support by browser - also called h2) the Connection
  must be created on top of SSL connection. For unencrypted connections (h2c)
  the Connection has to be created on top of a TCP stream.  
  On top of the connection multiple bidirectional streams can be established.
- `IStream` represents a single bidirectional HTTP/2 stream.  
  Each Request/Response pair in HTTP/2 is represented by a single Stream.

In the most common lifecycle a HTTP/2 client will create a new Stream on top
of the Connection, send all required headers for the Request through the Stream,
and then send optional data through the stream. The server will receive a
notification about a new create stream and invoke a request handler. The request
handler can then read the received headers, read all incoming data from the
stream and respond with headers and data.

Besides that there are some advanced features, e.g. both clients and servers
can also Cancel/Reset the stream during processing and both are able to send
trailing headers at the end of a Stream.

This library does not create the underlying TCP or SSL connections on server or
client side. This is the responsibility of the library user. However this
pattern allows to use the library on top of arbitrary IO stream types.

To create a `Connection` on top of IO streams the constructor of `Connection`
can be used:

```cs
var http2Connection = new Connection(new Connection.Options
{
    InputStream = inputStream,
    OutputStream = outputStream,
    IsServer = true,
    Settings = Settings.Default,
    StreamListener = AcceptIncomingStream,
});
```

The most important arguments here are:
- `InputStream`/`OutputStream`: The `Connection` will will use the IO streams
  to read data from the remote side and send data to the remote side. The
  connection will take ownership of these streams and close the `OutputStream`
  when done.
- `IsServer`: Specifies whether the local side of the `Connection` represents a
  HTTP/2 server (`true`) or client (`false`).
- `Settings`: The HTTP/2 settings that should be used. `Settings.Default`
  represents the default values from the HTTP/2 specification and should usually
  be a good choice.
- `StreamListener`: This is a callback function that will be invoked every time
  a new Stream is initiated from the remote side. This means it is currently
  only required for HTTP/2 server applications. An `IStream` instance which
  represents the initiated stream is passed to the callback function as an
  argument. The callback function should either return `true` if it wants to
  process the new stream or `false` otherwise. If the application wants to
  process the new stream it must use it in another `Task` and must not block the
  callback.

Besides that there are various optional arguments which allow further control
about the desired behavior.

After the `Connection` was set up it will handle all HTTP/2 protocol related
concerns up to the point where the underlying connection gets closed.

The `IStream` class allows to read and write headers and bytes for a single
HTTP/2 stream. The methods `ReadHeaders()` and `WriteHeaders()` allow to read
and write headers from a connection. The returned `ValueTask` from `ReadHeaders`
will only be fulfilled once headers from the remote side where received.
According to the HTTP/2 request/reponse lifecycle applications have to send
headers at the start of each Stream - which means it is not allowed to send
data before any headers have been sent. The received headers will contain the
pseudo-headers which are used in HTTP/2 for transmission of the HTTP method
(`:method`), status code (`:status`), etc. These are not automatically extracted
into seperated fields because `IStream` is used for requests and responses -
depending on whether the library is used for a client or server. However higher
level frameworks can easily extract the fields from the returned header lists.
This library will always validate received and sent headers for correctness.
This means a user of the library on the server side can rely on the fact that
the `:method`, `:scheme` and `:path` pseudo headers are preset and that after
the first real header field no pseudo header field follows.

After headers were sent for a Stream data can be written to the stream. As
`IStream` implements the ByteStream abstractions in this library, the data can
be written through the provided `WriteAsync` function. The returned `Task` will
only be fulfilled after the data could be sent through the underlying
connection. This will take the available flow control windows into account. The
stream can be closed from the by calling the `CloseAsync` function. Alternativly
the `endOfStream` parameter of `WriteHeaders` can be used for closing the Stream
without sending any data (e.g. for HTTP GET requests). The read and write
directions of HTTP/2 are fully independet. This means closing the local side of
the stream will still allow reading headers and data from the remote side. A
stream is only fully closed after all headers and data were consumed from the
remote side in addition to the local side having been closed. To read data from
the stream the `ReadAsync` function can be used. This function will signal the
end of stream if the stream was closed from the remote side. After a stream was
closed from the remote side trailers can be read through the `ReadTrailers`
function. To send trailers data must be transferred and instead of closing the
stream with `CloseAsync` or setting `endOfStream` to `true` the `WriteTrailers`
function must be used.

The `Cancel` function on the `IStream` can be used to reset the stream. This
will move the Stream into the Closed state if it wasn't Closed before. In order
to notify the remote side a RST_STREAM frame will be sent. All streams must be
either fully processed (read side fully consumed - write side closed) or
cancelled/reset by the application. Otherwise the Stream will be leaked. The
`Dispose` method on stream will also cancel the stream.

The following example shows the handling of a stream on the server side:

```cs
static async void HandleIncomingStream(IStream stream)
{
    try
    {
        // Read the headers
        var headers = await stream.ReadHeaders();
        var method = headers.First(h => h.Name == ":method").Value;
        var path = headers.First(h => h.Name == ":path").Value;
        // Print the request method and path
        Console.WriteLine("Method: {0}, Path: {1}", method, path);

        // Read the request body and write it to console
        var buf = new byte[2048];
        while (true)
        {
            var readResult = await stream.ReadAsync(new ArraySegment<byte>(buf));
            if (readResult.EndOfStream) break;
            // Print the received bytes
            Console.WriteLine(Encoding.ASCII.GetString(buf, 0, readResult.BytesRead));
        }

        // Send a response which consists of headers and a payload
        var responseHeaders = new HeaderField[] {
            new HeaderField { Name = ":status", Value = "200" },
            new HeaderField { Name = "content-type", Value = "text/html" },
        };
        await stream.WriteHeaders(responseHeaders, false);
        await stream.WriteAsync(new ArraySegment<byte>(
            Encoding.ASCII.GetBytes("Hello World!")), true);

        // Request is fully handled here
    }
    catch (Exception)
    {
        stream.Cancel();
    }
}
```

Current limitations
-------------------

The library currently faces the following limitations:
- Missing support for creating streams from client side.
- Missing support for manually sending a GOAWAY message to the remote.
- Missing support for reading the GOAWAY message received from a remote.
- Missing support for the push promises.
- Missing support for sending PING frames and waiting for the response.  
  However the library will respond to PING frames which are sent by the remote.
- Missing support for using base64 encoded SETTINGS data in case of connection
  upgrade from HTTP/1.1. This currently doesn't allow the h2c upgrade mechanism.
- Missing support for reading the remote SETTINGS from application side.
- The scheduling of outgoing DATA frames is very basic and relies mostly on flow
  control windows and the maximum supported frame size. It is currently not
  guaranteed that concurrent streams with equal flow control windows will get
  the same amount of bandwith.
  HTTP/2 priorization features are also not supported - however these are optional
  according to the HTTP/2 specification and may not be required for lots of
  applications.