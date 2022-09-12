### Abstract

This document is a proposal of a system allowing to connect to networks that
do not have open ports to the internet. This is a work in progress. It is
a common place to for sharing ideas and working on formal specifications.

Aforementioned system will consist of several processes running on different
machines communicating using a SOCKS5 protocol and another custom protocol
that will be described later. All of the network communication will happen
on the TCP layer (with later possibility to upgrade it to TLS).

### Definitions

*Agent* is a machine running on a closed network (that cannot accept connections
from the internet, but can send them). Agent is running a reverse proxy process
known as *agent's proxy*. *Routing server* (aka *the server*) is a process running
on another machine that is on the open network. It is used to route communication
between clients and agent. *Client* is another machine that is outside agent's
network. It communicates with the agent through the server. Client perhaps will be
running custom forward proxy server on their machine, known as *client's proxy*,
that will send their traffic to the server; although whether it is necessary is not
clear yet.

### System's overview

The goal of this system is to allow for clients to communicate with an agent that
is in the closed network. This communication will pass through routing server
that will send clients requests to the agent and return agent's responses to the
client. Because agent is on the closed network the server will use a mechanism
similar to "long polling" to communicate with an agent.

From the perspective of the client routing server is a SOCKS5 proxy. If SOCKS5
protocol alone will not be enough to handle this communication (authentication
and agent and client identification can be problematic as we do not want to
us an IP address as and unique id) then client will run their local forward proxy
that will from the client's side be a SOCKS5 proxy server and will translate
client's traffic using some custom protocol to include needed identifying
and authorizing information and send it to the server.

From the perspective of the agent the server will use a custom protocol to
communicate with it using a "long polling" mechanism. Since agent is on closed
network it cannot accept any connections from anyone, therefore each connection
between agent and the server must be initiated by an agent. Once agent connects
to the server it opens a channel for them to communicate over. It will be hanging
and not used until server has some request from one of the clients. Then through
this channel server will send it to agent, agent will through it's reverse proxy
send this request to it's network and when it gets a response from it's network
it sends it back to the server through the same connection that it opened. Once
server gets this response it sends it back to the client using SOCKS5 protocol.

In this system the server is responsible for keeping track of clients and agents,
assigning them unique ids, authorising clients to agents and keeping track of
socket addresses that agent allows for clients to connect to.

### Long polling proposals

Since an agent cannot know when the server will have a pending request from
some client it must always be ready to accept one and must always have at least
one open connection to the server. Since one connection must be responsible for
handling one users request (we could reuse the same connection to handle multiple
request but keeping track of which response is for which request will be *giant*
pain) agent should have an open "pool" of connections to help reduce latency.
The question is how an agent can know how many requests are awaiting on the server
side and how many open connections it needs to send to the server? Here are two
possible solutions.

First (and preferable) solution is to have beside a pool of connections one
"master" connection between an agent and the server. Over this master connection
the server can inform an agent about awaiting connections and possibly request
for more connections to be sent from an agent, when it has detected more client's
requests than are free waiting agent's polling connections. Communication over
this master socket will definitely be using some custom protocol, but in the best
case scenario communication over pool's connections will just us SOCKS5 protocol.

The second solution is to not have a master channel, but to send this kind of
information that would be send over a master channel in (custom) headers of
client's packets. This will probably make the protocol more complicated and
will mix logical synchronisation between agent and the server with transferring
data and increase size of network packets, so is not preferred.

Another question is should pool's connection be closed after it was used to send
client's request and agent's response, or should it be kept open for later
data transfers? This draft proposes closing a connection and (maybe) opening
a new one, as it will simplify data flow and status of each connection will be
better understood.

### Custom agent-server protocol

This draft does not define yet how this protocol should look, but it has to have
at least the following properties:
	- It must use TCP (or later TLS) as a layer of communication
	- It must support versioning, so it will be able to be changed in the future
	- It must use network byte-order (big endian) for all numbers
	- It must allow for the server to assign ids to clients and agents and send them
back to them
	- It must allow for agent to specify which local network's socket addresses are
open (and preferably for each clients)
	- It must be able to allow the server to request more connections from an agent
and for agent to confirm that these connections has been sent
