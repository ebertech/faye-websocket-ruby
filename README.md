# faye-websocket

* Travis CI build: [![Build
  status](https://secure.travis-ci.org/faye/faye-websocket-ruby.png)](http://travis-ci.org/faye/faye-websocket-ruby)
* Autobahn tests: [server](http://faye.jcoglan.com/autobahn/servers/),
  [client](http://faye.jcoglan.com/autobahn/clients/)

This is a robust, general-purpose WebSocket implementation extracted from the
[Faye](http://faye.jcoglan.com) project. It provides classes for easily
building WebSocket servers and clients in Ruby. It does not provide a server
itself, but rather makes it easy to handle WebSocket connections within an
existing [Rack](http://rack.rubyforge.org/) application. It does not provide
any abstraction other than the standard [WebSocket
API](http://dev.w3.org/html5/websockets/).

It also provides an abstraction for handling
[EventSource](http://dev.w3.org/html5/eventsource/) connections, which are
one-way connections that allow the server to push data to the client. They are
based on streaming HTTP responses and can be easier to access via proxies than
WebSockets.

Currently, the following web servers are supported, and can be accessed
directly or via HAProxy:

* [Goliath](http://postrank-labs.github.com/goliath/)
* [Puma](http://puma.io/)
* [Rainbows](http://rainbows.rubyforge.org/)
* [Thin](http://code.macournoyer.com/thin/)

Any web server that supports the `rack.hijack` API should also work.

The server-side socket can process
[draft-75](http://tools.ietf.org/html/draft-hixie-thewebsocketprotocol-75),
[draft-76](http://tools.ietf.org/html/draft-hixie-thewebsocketprotocol-76),
[hybi-07](http://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-07)
and later versions of the protocol. It selects protocol versions automatically,
supports both `text` and `binary` messages, and transparently handles `ping`,
`pong`, `close` and fragmented messages.


## Installation

```
$ gem install faye-websocket
```


## Handling WebSocket connections in Rack

You can handle WebSockets on the server side by listening for requests using
the `Faye::WebSocket.websocket?` method, and creating a new socket for the
request. This socket object exposes the usual WebSocket methods for receiving
and sending messages. For example this is how you'd implement an echo server:

```ruby
# app.rb
require 'faye/websocket'

App = lambda do |env|
  if Faye::WebSocket.websocket?(env)
    ws = Faye::WebSocket.new(env)

    ws.onmessage = lambda do |event|
      ws.send(event.data)
    end

    ws.onclose = lambda do |event|
      p [:close, event.code, event.reason]
      ws = nil
    end

    # Return async Rack response
    ws.rack_response

  else
    # Normal HTTP request
    [200, {'Content-Type' => 'text/plain'}, ['Hello']]
  end
end
```

This is a standard Rack app, so it can be run using a `config.ru` file.
However, so that incoming requests can be properly prepared to process
WebSocket connections, `Faye::WebSocket` needs to monkey-patch certain web
servers. If you're using one of these servers, you need to load an adapter:

* Goliath
* Rainbows! version < 4.5.0
* Thin

You load the adapter like so, passing either `goliath`, `rainbows` or `thin`.

```ruby
# config.ru
require './app'
Faye::WebSocket.load_adapter('thin')
run App
```

Note that under certain circumstances (notably a draft-76 client connecting
through an HTTP proxy), the WebSocket handshake will not be complete after you
call `Faye::WebSocket.new` because the server will not have received the entire
handshake from the client yet. In this case, calls to `ws.send` will buffer the
message in memory until the handshake is complete, at which point any buffered
messages will be sent to the client.

If you need to detect when the WebSocket handshake is complete, you can use the
`onopen` event.

If the connection's protocol version supports it, you can call `ws.ping()` to
send a ping message and wait for the client's response. This method takes a
message string, and an optional callback that fires when a matching pong
message is received. It returns `true` iff a ping message was sent. If the
client does not support ping/pong, this method sends no data and returns
`false`.

```ruby
ws.ping 'Mic check, one, two' do
  # fires when pong is received
end
```


## Using the WebSocket client

The client supports both the plain-text `ws` protocol and the encrypted `wss`
protocol, and has exactly the same interface as a socket you would use in a web
browser. On the wire it identifies itself as hybi-13.

```ruby
require 'faye/websocket'
require 'eventmachine'

EM.run {
  ws = Faye::WebSocket::Client.new('ws://www.example.com/')

  ws.onopen = lambda do |event|
    p [:open]
    ws.send('Hello, world!')
  end

  ws.onmessage = lambda do |event|
    p [:message, event.data]
  end

  ws.onclose = lambda do |event|
    p [:close, event.code, event.reason]
    ws = nil
  end
}
```


## Subprotocol negotiation

The WebSocket protocol allows peers to select and identify the application
protocol to use over the connection. On the client side, you can set which
protocols the client accepts by passing a list of protocol names when you
construct the socket:

```ruby
ws = Faye::WebSocket::Client.new('ws://www.example.com/', ['irc', 'amqp'])
```

On the server side, you can likewise pass in the list of protocols the server
supports after the other constructor arguments:

```ruby
ws = Faye::WebSocket.new(env, ['irc', 'amqp'])
```

If the client and server agree on a protocol, both the client- and server-side
socket objects expose the selected protocol through the `ws.protocol` property.
If they cannot agree on a protocol to use, the client closes the connection.


## WebSocket API

The WebSocket API consists of several event handlers and a method for sending
messages.

* <b>`onopen`</b> fires when the socket connection is established. Event has no
  attributes.
* <b>`onerror`</b> fires when the connection attempt fails. Event has no
  attributes.
* <b>`onmessage`</b> fires when the socket receives a message. Event has one
  attribute, <b>`data`</b>, which is either a `String` (for text frames) or an
  `Array` of byte-sized integers (for binary frames).
* <b>`onclose`</b> fires when either the client or the server closes the
  connection. Event has two optional attributes, <b>`code`</b> and
  <b>`reason`</b>, that expose the status code and message sent by the peer
  that closed the connection.
* <b>`send(message)`</b> accepts either a `String` or an `Array` of byte-sized
  integers and sends a text or binary message over the connection to the other
  peer.
* <b>`close(code, reason)`</b> closes the connection, sending the given status
  code and reason text, both of which are optional.
* <b>`protocol`</b> is a string (which may be empty) identifying the
  subprotocol the socket is using.


## Handling EventSource connections in Rack

EventSource connections provide a very similar interface, although because they
only allow the server to send data to the client, there is no `onmessage` API.
EventSource allows the server to push text messages to the client, where each
message has an optional event-type and ID.

```ruby
# app.rb
require 'faye/websocket'

App = lambda do |env|
  if Faye::EventSource.eventsource?(env)
    es = Faye::EventSource.new(env)
    p [:open, es.url, es.last_event_id]

    # Periodically send messages
    loop = EM.add_periodic_timer(1) { es.send('Hello') }

    es.onclose = lambda do |event|
      EM.cancel_timer(loop)
      es = nil
    end

    # Return async Rack response
    es.rack_response

  else
    # Normal HTTP request
    [200, {'Content-Type' => 'text/plain'}, ['Hello']]
  end
end
```

The `send` method takes two optional parameters, `:event` and `:id`. The
default event-type is `'message'` with no ID. For example, to send a
`notification` event with ID `99`:

```ruby
es.send('Breaking News!', :event => 'notification', :id => '99')
```

The `EventSource` object exposes the following properties:

* <b>`url`</b> is a string containing the URL the client used to create the
  EventSource.
* <b>`last_event_id`</b> is a string containing the last event ID received by
  the client. You can use this when the client reconnects after a dropped
  connection to determine which messages need resending.

When you initialize an EventSource with `Faye::EventSource.new`, you can pass
configuration options after the `env` parameter. Available options are:

* <b>`:retry`</b> is a number that tells the client how long (in seconds) it
  should wait after a dropped connection before attempting to reconnect.
* <b>`:ping`</b> is a number that tells the server how often (in seconds) to
  send 'ping' packets to the client to keep the connection open, to defeat
  timeouts set by proxies. The client will ignore these messages.

For example, this creates a connection that pings every 15 seconds and is
retryable every 10 seconds if the connection is broken:

```ruby
es = Faye::EventSource.new(es, :ping => 15, :retry => 10)
```

You can send a ping message at any time by calling `es.ping()`. Unlike
WebSocket the client does not send a response to this; it is merely to send
some data over the wire to keep the connection alive.


## Running your socket application

To use this library you must be using an EventMachine-based server; currently
Thin, Rainbows and Goliath are supported.


### Running the app with Thin

Thin can be started via the command line if you've set up a `config.ru` file
for your application:

```
$ thin start -R config.ru -p 9292
```

Or, you can use `rackup`. In development mode, this adds middlewares that don't
work with async apps, so you must start it in production mode:

```
$ rackup config.ru -s thin -E production -p 9292
```

It can also be started using the `Rack::Handler` interface common to many Ruby
servers. It must be run using EventMachine, and you can configure Thin further
in a block passed to `run`:

```ruby
require 'eventmachine'
require 'rack'
require 'thin'
require './app'

Faye::WebSocket.load_adapter('thin')

EM.run {
  thin = Rack::Handler.get('thin')

  thin.run(App, :Port => 9292) do |server|
    # You can set options on the server here, for example to set up SSL:
    server.ssl_options = {
      :private_key_file => 'path/to/ssl.key',
      :cert_chain_file  => 'path/to/ssl.crt'
    }
    server.ssl = true
  end
}
```


### Running the app with Puma

Puma has a command line interface for starting your application:

```
$ puma config.ru -p 9292
```

You do not need to call `Faye::WebSocket.load_adapter` to work with Puma but
you must use at least version 2.0.0 of the `puma` gem.


### Running the app with Rainbows

You can run your `config.ru` file from the command line. Again, `Rack::Lint`
will complain unless you put the application in production mode.

```
$ rainbows config.ru -c path/to/rainbows.conf -E production -p 9292
```

Rainbows also has a Ruby API for starting a server:

```ruby
require 'rainbows'
require './app'

rackup = Unicorn::Configurator::RACKUP
rackup[:port] = 9292
rackup[:set_listener] = true
options = rackup[:options]
options[:config_file] = 'path/to/rainbows.conf'

server = Rainbows::HttpServer.new(App, options)

# This is non-blocking; use server.start.join to block
server.start
```

If you're using version 4.4 or lower of Rainbows, you need to run it with the
EventMachine backend and enable the adapter. Put this in your `rainbows.conf`
file:

```ruby
# rainbows.conf
Rainbows! do
  use :EventMachine
end
```

And make sure you load the adapter in your application:

```ruby
Faye::WebSocket.load_adapter('rainbows')
```


### Running the app with Goliath

Goliath can be made to run arbitrary Rack apps by delegating to them from a
`Goliath::API` instance. A simple server looks like this:

```ruby
require 'goliath'
require './app'
Faye::WebSocket.load_adapter('goliath')

class EchoServer < Goliath::API
  def response(env)
    App.call(env)
  end
end
```

`Faye::WebSocket` can also be used inline within a Goliath app:

```ruby
require 'goliath'
require 'faye/websocket'
Faye::WebSocket.load_adapter('goliath')

class EchoServer < Goliath::API
  def response(env)
    ws = Faye::WebSocket.new(env)

    ws.onmessage = lambda do |event|
      ws.send(event.data)
    end

    ws.rack_response
  end
end
```


## License

(The MIT License)

Copyright (c) 2010-2013 James Coglan

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the 'Software'), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

