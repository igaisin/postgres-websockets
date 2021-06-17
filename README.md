# postgres-websockets

![CI](https://github.com/diogob/postgres-websockets/actions/workflows/ci.yml/badge.svg)
[![Hackage Matrix CI](https://matrix.hackage.haskell.org/api/v2/packages/postgres-websockets/badge)](https://matrix.hackage.haskell.org/package/postgres-websockets)


postgres-websockets is a [middleware](https://hackage.haskell.org/package/wai) that adds websockets capabilites on top of [PostgreSQL](https://www.postgresql.org)'s asynchronous notifications using LISTEN and NOTIFY commands.
The project was largely inspired and originaly designed for use with [PostgREST](https://github.com/begriffs/postgrest).

postgres-websockets allows you to:
 * Open websockets with multiple channels and exchange messages with the database asynchronously.
 * Send messages to that websocket so they become a [NOTIFY](https://www.postgresql.org/docs/current/static/sql-notify.html) command in a PostgreSQL database.
 * Receive messages sent to any database channel though a websocket.
 * Authorize the use of channels using a [JWT](https://jwt.io) issued by another service or by another PostgREST endpoint.
 * Authorize read-only, write-only, or read and write websockets.

## Running the server

### Quickstart using docker-compose
The `docker-compose.yml` present in the repository will start a PostgreSQL database alongside a postgres-websockets and a pg-recorder.
To try it out you will need [Docker](https://www.docker.com/) installed and [git](https://git-scm.com) to clone this repository.

```bash
git clone https://github.com/diogob/postgres-websockets.git
cd postgres-websockets
docker-compose up
```

### Building from source
To build the project I recommend the use of [Stack](http://docs.haskellstack.org/en/stable/README/).
You also need to have [git](https://git-scm.com) installed to download the source code.
Having installed stack the following commands should install `postgres-websockets` into your `~/.local/bin` directory:

```bash
git clone https://github.com/diogob/postgres-websockets.git
cd postgres-websockets
stack setup
stack build
```

If you have any problems processing any Postgres related library on a Mac, try installing [Postgres.app](http://postgresapp.com/).

After the build you should be able to run the server using `~/.local/bin/postgres-websockets` (you can add `~/.local/bin` to your PATH variable):

To run the example bellow you will need a PostgreSQL server running on port 5432 of your localhost.
```bash
PGWS_DB_URI="postgres://localhost:5432/postgres" PGWS_JWT_SECRET="auwhfdnskjhewfi34uwehdlaehsfkuaeiskjnfduierhfsiweskjcnzeiluwhskdewishdnpwe" ~/.local/bin/postgres-websockets
postgres-websockets <version> / Connects websockets to PostgreSQL asynchronous notifications.
Listening on port 3000
```

 You can also use the provided [sample-env](./sample-env) file to export the needed variables:
```bash
source sample-env && ~/.local/bin/postgres-websockets
```
After running the above command, open your browser on http://localhost:3000 to see an example of usage.

The sample config file provided in the [sample-env](https://github.com/diogob/postgres-websockets/tree/master/sample-env) file comes with a jwt secret just for testing and is used in the sample client.
Note that the `sample-env` points to `./database-uri.txt` to load the URI from an external file. This is determined by the use of `@` as a prefix to the value of the variable `PGWS_DB_URI`. 
This is entirely optional and the URI could be exported directly as `PGWS_DB_URI` without using the prefix `@`.
You will find the complete sources for the example under the folder [client-example](https://github.com/diogob/postgres-websockets/tree/master/client-example).
To run the server without giving access to any static files one can unser the variable `PGWS_ROOT_PATH`.

## Opening connections

To open a websocket connection to the server there are two possible request formats.

1. Requesting a channel and giving a token

When you request access to a channel called `chat` the address of the websockets will look like:
```
ws://chat/eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtb2RlIjoicncifQ.QKGnMJe41OFZcjz_qQSplmWAmVd_hmVjijKUNoJYpis
```
When the token contains a `channels` claim, the value of that claim should be a list of allowed channels.
Any requested channel not set in that claim will result in an error opening the connection. 
Tokens without the `channels` claim (like the example above) are capable of opening connections to any channel, so be careful when issuing those.


2. Giving only the token

When you inform only the token on the websocket address, the `channels` claim must be present.
In this case all channels present in the claim will be available simultaneously in the same connection.
This is useful for clients that need to monitor or boradcast a set of channels, being more convenient than managing multiple websockets.
The address will look like:
```
ws://eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtb2RlIjoicnciLCJjaGFubmVsIjoiY2hhdCJ9.fEm6P7GHeJWZG8OtZhv3H0JdqPljE5dainvsoupM9pA
```

To use a secure socket (`wss://`) you will need a proxy server like nginx to handle the TLS layer. Some services (e.g. Heroku) will handle this automatially.

## Receiving messages from the browser

Every message received from the browser will be in JSON format as:
```javascript
{
  "event": "WebsocketMessage",
  "channel": "destination_channel",
  "payload": "message content",
  "claims": { "message_delivered_at": 0.0, "a_custom_claim_from_the_jwt": "your_custom_value" }
}
```

Where `claims` contain any custom claims added to the JWT with the added `message_delivered_at` which marks the timestamp in unix format of when the message was processed by postgres-websockets just before being sent to the database.
Also `channel` contains the channel used to send the message, this should be used to send any messages back to that particular client.
Finally `payload` contain a string with the message contents.

A easy way to process messages received asynchronously is to use [pg-recorder](https://github.com/diogob/pg-recorder) with some custom stored procedures.
For more options on notification processing check the [PostgREST documentation on the topic](https://postgrest.com/en/v4.3/intro.html#external-notification).

## Sending messages to the browser

To send a message to a particular channel on the browser one should notify the postgres-websockets listener channel and pass a JSON object containing the channel and payload such as:
```sql
SELECT pg_notify(
  'postgres-websockets-listener',
  json_build_object('event', 'WebsocketMessage', 'channel', 'chat', 'payload', 'test')::text
);
```

Where `postgres-websockets-listener` is the database channel used by your instance of postgres-websockets and `chat` is the channel where the browser is connected (the same issued in the JWT used to connect).

## Monitoring Connections

To monitor connection opening one should set the variable `PGWS_META_CHANNEL` which will enable the meta-data messages generation in the server on the channel name specified.
For instamce, if we use the configuration in the [sample-env](./sample-env) we will see messages like the one bellow each time a connection is estabilished (only after the JWT is validated).

```javascript
{"event":"ConnectionOpen","channel":"server-info","payload":"server-info","claims":{"mode":"rw","message_delivered_at":1.602719440727465893e9}}
```

You can monitor these messages on another websocket connection with a proper read token for the channel `server-info` or also having an additional database listener on the `PGWS_LISTEN_CHANNEL`.

## Recovering from listener database connection failures

The database conneciton used to wait for notification where the `LISTEN` command is issued can cause problems when it fails. To prevent these problem from completely disrupting our websockets server there are two ways to configure postgres-websockets:

* Self healing connection - postgres-websockets comes with a connection supervisor baked in. You just need to set the configuration `PGWS_CHECK_LISTENER_INTERVAL` to a number of milliseconds that will be the maximum amount of time losing messages. There is a cost for this since at each interval an additional SELECT query will be issued to ensure the listener connection is still active. If the connecion is not found the connection thread will be killed and respawned. This method has the advantage of keeping all channels and websocket connections alive while the database connection is severed (although messages will be lost).
* Using external supervision - you can also unset `PGWS_CHECK_LISTENER_INTERVAL` and postgres-websockets will try to shutdown the server when the database connection is lost. This does not seem to work in 100% of the cases, since in theory is possible to have the database connection closed and the producer thread lingering. But in most cases it should work and some external process can then restart the server. The downside is that all websocket connections will be lost.