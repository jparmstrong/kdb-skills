# kdb/q IPC Reference

## Opening and closing connections

```q
/ Connect to localhost port 5001
h:hopen `::5001

/ Connect with user:password
h:hopen `:localhost:5001:user:pass

/ Connect to remote host
h:hopen `:server.example.com:5001

/ Always close when done
hclose h
```

`hopen` returns an integer handle. Reuse the handle for multiple queries — opening per query is expensive.

## Sending queries

### Synchronous (blocking)

```q
/ String form
h"select from trade where sym=`AAPL"

/ Parse tree form (preferred — avoids string parsing overhead)
h(`.myns.myfunc; arg1; arg2)

/ Pass a local function to the server
f:{x*x}
h(f; 10)     / runs f on server with arg 10

/ Named args via dict
h(`.myns.load; `trade; 2024.01.15)
```

The client blocks until the server returns a result or error.

### Asynchronous (non-blocking)

```q
/ Use negative handle for async
neg[h]"insert[`trade;(enlist .z.p;`AAPL;100.0;1000i)]"

/ Async with parse tree
neg[h](`.myns.publish; data)

/ Flush queued async messages
neg[h][]
```

Async messages are queued, not sent immediately. `neg[h][]` flushes. The client does not wait for a response.

### One-shot (temporary connection)

```q
`:localhost:5001"1+1"    / connect, query, disconnect automatically
```

Convenient for scripts but expensive for repeated calls.

## Server-side callbacks (.z namespace)

Override these to handle incoming messages:

```q
/ Synchronous message handler (default: value)
.z.pg:{[msg]
  -1 "sync from ",(string .z.a),": ",-3!msg;
  value msg          / execute and return result
  }

/ Asynchronous message handler (default: value, no return)
.z.ps:{[msg]
  value msg
  }

/ On open connection
.z.po:{[h]
  -1 "connection from: ",(string .z.a);
  }

/ On close connection
.z.pc:{[h]
  -1 "disconnected: ",(string h);
  }

/ On WebSocket message
.z.ws:{[msg]
  neg[.z.w] msg   / echo back
  }
```

Key `.z` variables in handlers:

| Variable | Meaning |
|----------|---------|
| `.z.w` | Handle of current connection |
| `.z.a` | IP address of caller |
| `.z.u` | Username of caller |
| `.z.i` | PID of caller process |

## Deferred synchronous pattern

For high-throughput servers: client sends async, server responds async.

```q
/ Server side:
.z.ps:{[msg]
  result:value msg;
  neg[.z.w](`.callback; result)  / respond to client
  }

/ Client side:
neg[h](`.myQuery; args)  / send async
/ ... client's .z.ps receives the response
```

## Broadcast

Send the same message to multiple handles efficiently (serialize once):

```q
/ Serialize once, send to many
msg:-8!data                   / serialize
{h msg} each handles           / naive: serializes N times

/ Better: use -25! for single serialization
-25!(handles; data)            / internal broadcast function
```

## Subscribing / ticker plant pattern

Standard KDB tick pattern:

```q
/ Subscribe to all syms on all tables
.u.sub[`;`]

/ Subscribe to specific table and syms
.u.sub[`trade;`AAPL`GOOG]

/ Handle incoming updates
upd:{[t;x] t insert x}        / append to table t
```

## Error handling over IPC

```q
/ Protected evaluation — catches errors from remote
@[h; "bad query"; {-1 "Error: ",x}]

/ Or catch in a protected eval on local side
.[h; enlist "bad query"; {-1 "Remote error: ",x}]
```

## Serialization

```q
/ Serialize (byte vector)
-8!data

/ Deserialize
-9!bytes

/ Size of serialized form
count -8!data

/ Compress before sending
-18!data   / compress
-19!bytes  / decompress
```

## Starting a listening process

```q
/ From command line
q myprocess.q -p 5001

/ Or within q
\p 5001    / set port (process is now listening)

/ Check current port
\p
```

## Common IPC patterns

```q
/ Retry on disconnect
getHandle:{
  if[not h:@[hopen;`:localhost:5001;0]; '"Connection failed"];
  h}

/ Connection pool (simplified)
handles:hopen each `::5001`::5002`::5003
results:handles@\:"select from trade"  / each-right query

/ Async fan-out, sync collect
{neg[x] query} each handles   / send async
neg[handles]@\:[]              / flush each
/ ... handle responses in .z.ps
```

## Security considerations

- Use `.z.pw` to validate user/password: ``.z.pw:{[u;p] p~passwd[u]}``
- Use `.z.pg` / `.z.ps` to restrict which queries can run
- Never expose raw `value` to untrusted clients in production
- Use `.z.a` to whitelist by IP if needed
