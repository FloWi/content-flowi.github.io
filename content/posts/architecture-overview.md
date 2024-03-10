+++
title = "Spacetraders Agent written in scala - Architecture overview"
date = "2024-03-10T12:15:49+01:00"
author = "Florian Witteler"
authorTwitter = "flowi" #do not include @
cover = ""
tags = ["spacetraders", "scala", "scala-js", "outwatch"]
keywords = ["", ""]
description = ""
showFullContent = false
tableOfContents = true
+++

# Overview

## Infrastructure

My go-to setup for running infrastructure services locally is docker-compose. I'm using postgres as a database and grafana for some dashboards.

- docker-compose setup for infrastructure
    - postgres
    - grafana

Since I also developed a dashboard for the agents and wanted to reuse some helpers, I added profiles to docker-compose. If I want to start the stack for the
agent, I run this command:

```shell
docker compose --profile agent up
```

## agent architecture

I like coding in a functional style and use [cats-effect](https://typelevel.org/cats-effect/) as the runtime effect system. I had trouble with the openapi
generator for scala, since it didn't work in some cases or didn't produce code that matches my coding style.

Luckily the folks at Disney created a nice ecosystem around smithy that worked very well for code generation.
I'm using [smithy-translate](https://github.com/disneystreaming/smithy-translate) to convert the openapi-spec into a smithy model (with minor fixes) and from
there I use [smithy4s](https://github.com/disneystreaming/smithy4s) to generate the client.
Shout out to the authors (especially @Baccata) who were very helpful in their discord channel when I ran into problems.

The library is awesome and generates code for an [http4s-client](https://http4s.org/) that is easily extensible:

- adding authorization via bearer-token
- enabling request logging
- adding rate-limiting with priority [upperbound](https://github.com/SystemFw/upperbound)
- easy to fix
    - had to change the behavior of posting an empty request; client was sending `null` but st-server didn't like that

### libraries used

| lib              | description                                                                                 |
|------------------|---------------------------------------------------------------------------------------------|
| smithy-translate | tool for converting openapi-sepc to smithy model                                            |
| smithy4s         | plugin for generating http4s client/server from smithy model                                |
| http4s           | http client/server                                                                          |
| cats-effect      | runtime effect system                                                                       |
| upperbound       | rate limiting for cats-effect IO                                                            |
| circe            | json (de)serialization                                                                      |
| enumeratum       | better enums for scala                                                                      |
| monocle          | optics library for easier manipulation of nested properties inside immutable datastructures |
| flyway           | database migration                                                                          |
| doobie           | functional JDBC layer                                                                       |
| fs2              | functional streaming                                                                        |

## bootstrapping of agent

The startup of the agent happens with the unregistered instance of the client. It doesn't use a bearer-token.

### database migration

When I start the agent, the first thing I check is the [status endpoint](https://api.spacetraders.io/v2/). I take the reset-date from its response, parse it and
use it as a schema name in postgres.
e.g. this response

```json
{
  "status": "SpaceTraders is currently online and available to play",
  "version": "v2.1.5",
  "resetDate": "2024-02-25"
}
```

becomes `reset_2024_02_25`.

I use flyway to migrate my database schema (it also creates one if it doesn't exist). From there I have a schema ready to be used and instantiate a hikari
connection-pool with the connectionstring.

```scala
def transactorResource(connectionString: String,
  username: String,
  password: String,
  schema: String): Resource[IO, HikariTransactor[IO]] =
  for {
    ce <- ExecutionContexts.fixedThreadPool[IO](32)
    xa <- HikariTransactor
      .newHikariTransactor[IO](
        "org.postgresql.Driver",
        connectionString,
        username,
        password,
        ce,
      )
      .evalTap(tx =>
        tx.configure { ds =>
          Async[IO].delay {
            ds setSchema schema
          }
        }
      )
  } yield xa
```

After that step I modify the readonly grafana user to have that schema as default in its search path.

```sql
-- Grant usage and all privileges on the schema to the user

GRANT USAGE ON SCHEMA reset_2024_02_25 TO grafana_user;

GRANT SELECT ON ALL TABLES IN SCHEMA reset_2024_02_25 TO grafana_user;

-- Grant all privileges on new tables in the schema to the user
ALTER DEFAULT PRIVILEGES IN SCHEMA reset_2024_02_25
    GRANT ALL ON TABLES TO grafana_user;

-- Set the default search path to the new schema
ALTER USER grafana_user SET search_path TO reset_2024_02_25;
```

That way I don't need to update the grafana dashboard with new connectionstrings.

### registering my agent if necessray

On startup I check my `registration` table to see if I already registered my agent and use its token. If not I use the infos from env vars to register. I added
a 10s delay before registering, because I forgot to change some settings in the past ;-)

```scala
IO.println(
  s"""Registering agent with these details:
     |  Symbol: ${registrationInputBody.symbol}
     |  E-Mail: ${registrationInputBody.email}
     |  Faction: ${registrationInputBody.faction}
     |
     |You have 10s to cancel if you want to change anything.""".stripMargin) *>
  IO.sleep(10.seconds) /* *> ... */

```


## background collection of "big" data

In my codebase I use two different clients that talk to the st-servers. One with a high priority that controls the ships or loads essential data and another one
with a low priority that collects data in the background (like systems, waypoints, jump-gates etc.)

The low priority client is used in a streaming context (with fs2) to prevent excessive memory usage and provide back-pressure. It's also straightforward to ingest responses from the paginated endpoints.

Here's an example in where I collect the waypoints of a system:

```scala
def loadSystemWaypointsPaginated(systemSymbol: String): Stream[IO, GetSystemWaypoints200] = {
  Stream.eval(getListWaypointsPageIO(systemSymbol, PaginationInput(page = Some(1), limit = Some(20))))
    .flatMap { first =>
      val rest = PaginationHelper.calcRestRequests(first.body.meta)
      val restStream: Stream[IO, GetSystemWaypoints200] = Stream
        .iterable(rest)
        .lift[IO]
        .evalMap(page => getListWaypointsPageIO(systemSymbol, page))

      Stream(first) ++ restStream
    }
}
```

- collect first page
- take `Meta` object from response of the first page to
- calculate the numbers of the remaining pages to create a stream
- concat the first page with the rest of the stream



##        

