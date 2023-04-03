+++
title = "Example of Domain Driven Design"
weight = 1
order = 1
date = 2023-04-03
insert_anchor_links = "right"
[taxonomies]
tags = ["rust", "DDD", "event-sourcing", "cqrs"]
+++

Code can be found at: [github.com/aurelien-clu/example-ddd-es/rust-cqrs](https://github.com/aurelien-clu/example-ddd-es/tree/main/rust-cqrs)

## Our objectives

Let's create an API controlling a `Plane`. We want to be able to:

- fly it
- land it
- change its position
- track for the current or previous journey, all past positions

For such simple needs, a `CRUD API` would suffice but for the sake of learning we will use [DDD](https://en.wikipedia.org/wiki/Domain-driven_design), [event-sourcing](https://en.wikipedia.org/wiki/Event-driven_architecture) & [CQRS](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation#Command_Query_Responsibility_Segregation).

## Event storming

> Event storming is a workshop-based method to quickly find out what is happening in the domain of a software program. Compared to other methods it is extremely lightweight and intentionally requires no support by a computer. The result is expressed in sticky notes on a wide wall.

[wikipedia.org/Event storming](https://en.wikipedia.org/wiki/Event_storming)


We will follow the following steps:

1. Collect domain events
2. Refine domain events
3. Track causes
4. Find aggregates

We end up with the following:

<img src="https://cluzeau.pro/ddd-example-plane.jpg" alt= "event storming" width="30%" height="30%"/>

*Graph made using: [miro.com/miroverse/event-storming/](https://miro.com/miroverse/event-storming/)*

Notes:

- Event storming should be done with domain experts, I am not an expert in aviation, thus this is incomplete but sufficient for this example :)
- *OnGround* event is not necessarily following *PlaneRegistered* but happens at the same time. It makes more sense to have an event describing that the plane is on ground without knowing whether the plane has flew before or not.

## Let's code it

Below you will find code samples, full code can be found here: [github.com/aurelien-clu/example-ddd-es/rust-cqrs](https://github.com/aurelien-clu/example-ddd-es/tree/main/rust-cqrs).

### Which library?

We want a `rust` library to help us with `DDD`, `event-sourcing` & `CQRS`. [github.com/serverlesstechnology/cqrs](https://github.com/serverlesstechnology/cqrs) fits our needs.

Reasons to use it:

- great documentation: [doc.rust-cqrs.org/](https://doc.rust-cqrs.org/)
- ease of use

Reasons not to use it:

- aggregate ids have to be `Strings`, we could want to use `UUID`
- maybe less feature complete than [github.com/get-eventually/eventually-rs](https://github.com/get-eventually/eventually-rs)
- not `wasm` if you have reasons to need this, then go with [github.com/thalo-rs/thalo](https://github.com/thalo-rs/thalo)

### Setup

```bash
.
├── Cargo.toml
├── crates
│   ├── domain-plane
│   └── server
├── db
│   └── init.sql
├── docker-compose.yml
└── test
    └── test_api_plane.sh
```

We need to have a crate for our `plane domain` and another for our `server`.
By separating `domain` from `server` (and other `infrastructure` concerns we adhere to [wikipedia.org/Hexagonal architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)))

#### **Cargo.toml**

```toml
[workspace]
members = ["crates/*"]
```

#### **crates/**

```bash
cargo new crates/domain-plane --lib
cargo new crates/server
```

#### **docker-compose.yml**

To store our events and our `queries` (views on our aggregate) we will need a database, [github.com/serverlesstechnology/cqrs](https://github.com/serverlesstechnology/cqrs) works with `PostgreSQL` & few other `databases`. `PostgreSQL` being my go to database, we will use it. (it is also the one in the demo project, let's keep it simple)

```yaml
version: '3.1'

services:
  cqrs-postgres-db:
    image: postgres
    restart: always
    ports:
      - 5432:5432
    environment:
      POSTGRES_DB: demo
      POSTGRES_USER: demo_user
      POSTGRES_PASSWORD: demo_pass
    volumes:
      - './db:/docker-entrypoint-initdb.d'
```

#### **db/init.sql**

We need to store our events and create a database user:

```sql
CREATE TABLE events
(
    aggregate_type text                         NOT NULL,
    aggregate_id   text                         NOT NULL,
    sequence       bigint CHECK (sequence >= 0) NOT NULL,
    event_type     text                         NOT NULL,
    event_version  text                         NOT NULL,
    payload        json                         NOT NULL,
    metadata       json                         NOT NULL,
    PRIMARY KEY (aggregate_type, aggregate_id, sequence)
);

CREATE USER demo_user WITH ENCRYPTED PASSWORD 'demo_pass';
GRANT ALL PRIVILEGES ON DATABASE postgres TO demo_user;
```

### Events

From our `event storming` session, we need to have the following events:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum Event {
    Registered { registration_id: String },
    OnGround,
    TookOff,
    Landed,
    PositionedAt {
        latitude: f64,
        longitude: f64,
        altitude: usize,
    },
}
```

they will be serialized as `json` in the database (column `payload`) thus we need to make them serializable.

### Commands

And also the following commands:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub enum Command {
    Register {
        registration_id: String,
    },
    UpdatePosition {
        latitude: f64,
        longitude: f64,
        altitude: usize,
    },
    TakeOff,
    Land,
}
```

They will be `Deserialized` from `http` queries, so it is not needed by our `domain-plane` crate but will be useful by the `server` crate.

### Errors

Commands may not always return successfully, and some errors can be expected and returned.
We create an `enum` with all possible errors and define a message associated to each (`#[error("<MSG>")]`).

Usually this will be updated while implementing the aggregate. To fasten the reading, here are all of our error cases.

```rust
#[derive(thiserror::Error, Clone, Debug, PartialEq)]
pub enum Error {
    #[error("Unable to take off in curent state")]
    CannotTakeOff,
    #[error("Unable to land in current state")]
    CannotLand,
    #[error("Cannot register again, identification is immutable")]
    AlreadyRegistered,
}
```

Note: [docs.rs/thiserror/latest/thiserror/](https://docs.rs/thiserror/latest/thiserror/) reduces the boilerplate around error definitions.

### Aggregate

#### **State**

Let's define our `Plane` state. We want it to have:

- an identification (relating to its domain -> [wikipedia.org/Aircraft_registration](https://en.wikipedia.org/wiki/Aircraft_registration))
- a last known position
- a flying status

```rust
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub struct Position {
    pub latitude: f64,
    pub longitude: f64,
    pub altitude: usize,
}

#[derive(Serialize, Deserialize, Default, Debug, Clone, PartialEq)]
pub enum Status {
    #[default]
    OnGround,
    InAir,
}

#[derive(Serialize, Deserialize)]
pub struct Plane {
    registration_id: String,
    last_position: Option<Position>,
    status: Status,
}

impl Default for Plane {
    fn default() -> Self {
        Self {
            registration_id: "".to_string(),
            last_position: None,
            status: OnGround,
        }
    }
}
```

We then need to be able to build a `Plane` state based on past events:

```rust
fn apply(&mut self, event: Self::Event) {
    match event {
        Event::Registered { registration_id } => self.registration_id = registration_id,
        Event::PositionedAt {
            latitude,
            longitude,
            altitude,
        } => {
            let p = Position {
                latitude,
                longitude,
                altitude,
            };
            self.last_position = Some(p);
        }
        Event::TookOff => self.status = Status::InAir,
        Event::Landed => self.status = Status::OnGround,
        Event::OnGround => self.status = Status::OnGround,
    }
}
```

#### **Command handler**

We have a `Plane` that can be constructed with past events but we have no way of creating events.

That where the `handle` function comes in, it will translate successful commands into events:

```rust
async fn handle(
    &self,
    command: Self::Command,
    _services: &Self::Services,
) -> Result<Vec<Self::Event>, Self::Error> {
    match command {
        Command::Register { registration_id } => {
            if self.registration_id != "" {
                return Err(Error::AlreadyRegistered);
            }
            Ok(vec![Event::Registered { registration_id }, Event::OnGround])
        }
        Command::UpdatePosition {
            latitude,
            longitude,
            altitude,
        } => Ok(vec![Event::PositionedAt {
            // here we should validate that coordinates are valid
            latitude,
            longitude,
            altitude,
        }]),
        Command::TakeOff => {
            if self.status == Status::OnGround {
                // here we should call the TowerControl service to ensure we can takeoff
                Ok(vec![Event::TookOff])
            } else {
                Err(Error::CannotTakeOff)
            }
        }
        Command::Land => {
            if self.status == Status::InAir {
                // here we should call the TowerControl service to ensure we can land
                Ok(vec![Event::Landed])
            } else {
                Err(Error::CannotLand)
            }
        }
    }
}
```

[rust-cqrs/crates/domain-plane/src/domain/aggregate.rs](https://github.com/aurelien-clu/example-ddd-es/blob/main/rust-cqrs/crates/domain-plane/src/domain/aggregate.rs)

#### **Services**

Our aggregate `Plane` can do some things on its own but for others it should be working with a `Control Tower` service.

In this example, we are not going to implement a "real" or a mocked version.

Know that we could define a `trait` within the `domain-plane` crate and have a `svc-tower-control` crate, for instance, implementing the `trait`. It would for instance ensure that no other planes are currently landing or departing whenever a plane asks to land or take off.

Services may be covered in future post, stay tuned!

#### **Tests**

How to ensure that everything is working properly? Let's test the aggregate with the following:

- a plane can be registered
- a plane can update its position
- a plane can take off if it is on ground
- a plane can land if it is airborne
- a plane cannot land if already on ground
- a plane cannot take off if already airborne
- a plane cannot register once it is registered

Example of a test, following [wikipedia.org/Behavior driven development](https://en.wikipedia.org/wiki/Behavior-driven_development):

```rust
#[test]
fn test_a_plane_should_land() {
    let past = vec![
        Event::Registered {
            registration_id: "F-TEST".to_string(),
        },
        Event::OnGround,
        Event::TookOff,
    ];
    let command = Command::Land;
    let expected = vec![Event::Landed];
    let services = ();
    PlaneTestFramework::with(services)
        .given(past)
        .when(command)
        .then_expect_events(expected);
}
```

[rust-cqrs/crates/domain-plane/src/domain/tests.rs](https://github.com/aurelien-clu/example-ddd-es/blob/main/rust-cqrs/crates/domain-plane/src/domain/tests.rs)

We verify everything works as expected with:

```bash
cargo test
```

### Queries

> track for the current or previous journey, all past positions

With `Command Query Responsibility Segregation` (CQRS) we should access past events with a "Query".

With our chosen library this means that we build a query over past event for our aggregate and as events happen, we will update the state of our query.

Our `CurrentJourneyView` will track in its state the past `positions` so we don't need to access directly the event log whenever we want this information, we will look up our new query table (cf below).

```rust
#[derive(Debug, Default, Serialize, Deserialize)]
pub struct CurrentJourneyView {
    registration_id: String,
    status: Status,
    positions: Vec<Position>,
}

pub type CurrentJourneyQuery =
    GenericQuery<PostgresViewRepository<CurrentJourneyView, Plane>, CurrentJourneyView, Plane>;

impl View<Plane> for CurrentJourneyView {
    fn update(&mut self, event: &EventEnvelope<Plane>) {
        match &event.payload {
            Event::Registered { registration_id } => self.registration_id = registration_id.clone(),
            Event::OnGround => self.status = Status::OnGround,
            Event::TookOff => {
                self.status = Status::InAir;
                self.positions.clear();
            }
            Event::Landed => self.status = Status::OnGround,
            Event::PositionedAt {
                latitude,
                longitude,
                altitude,
            } => self.positions.push(Position {
                latitude: *latitude,
                longitude: *longitude,
                altitude: *altitude,
            }),
        }
    }
}
```

[rust-cqrs/crates/domain-plane/src/queries/current_journey.rs](https://github.com/aurelien-clu/example-ddd-es/blob/main/rust-cqrs/crates/domain-plane/src/queries/current_journey.rs)

Let's update the `init.sql` with our new query:


```sql
CREATE TABLE plane_current_journey_query
(
    view_id text                        NOT NULL,
    version           bigint CHECK (version >= 0) NOT NULL,
    payload           json                        NOT NULL,
    PRIMARY KEY (view_id)
);
```

### Server

Let's use [github.com/tokio-rs/axum](https://github.com/tokio-rs/axum) for our server:

```rust
#[tokio::main]
async fn main() {
    let pool = default_postgress_pool("postgresql://demo_user:demo_pass@localhost:5432/demo").await;

    // we setup and get ownership of our aggregate and query
    let (cqrs, current_journey_query) = domain_plane::config::cqrs_framework(pool);

    let router = Router::new()
        .route(
            "/plane/:registration_id",
            get(routes::plane::query_handler).post(routes::plane::command_handler),
        )
        // we use axum extensions to provide:
        .layer(Extension(cqrs)) // - our aggregate to our command_handler
        .layer(Extension(current_journey_query)); // - our current_journey_query to our query_handler

    axum::Server::bind(&"0.0.0.0:3030".parse().unwrap())
        .serve(router.into_make_service())
        .await
        .unwrap();
}
```

[rust-cqrs/crates/server/src/main.rs](https://github.com/aurelien-clu/example-ddd-es/blob/main/rust-cqrs/crates/server/src/main.rs)

#### **Queries**

We lookup our query for the given `registration_id` and return the response as `json` or an error.

```rust
pub async fn query_handler(
    Path(registration_id): Path<String>,
    Extension(view_repo): Extension<Arc<PostgresViewRepository<CurrentJourneyView, Plane>>>,
) -> Response {
    let view = match view_repo.load(&registration_id).await {
        Ok(view) => view,
        Err(err) => {
            return (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()).into_response();
        }
    };
    match view {
        None => StatusCode::NOT_FOUND.into_response(),
        Some(account_view) => (StatusCode::OK, Json(account_view)).into_response(),
    }
}
```

#### **Commands**

We receive a command as `json` and apply it to the aggregate matching the `registration_id`.

Commands don't return data (cf `CQRS`) but a simple `204 NO CONTENT` successful response or an error.

```rust
pub async fn command_handler(
    Path(registration_id): Path<String>,
    Extension(cqrs): Extension<Arc<PostgresCqrs<Plane>>>,
    MetadataExtension(metadata): MetadataExtension,
    Json(command): Json<Command>,
) -> Response {
    match cqrs
        .execute_with_metadata(&registration_id, command, metadata)
        .await
    {
        Ok(_) => StatusCode::NO_CONTENT.into_response(),
        Err(err) => (StatusCode::BAD_REQUEST, err.to_string()).into_response(),
    }
}
```


### Tests

Let's see if our `API` works properly!

```bash
RANDOM=$$
TEST_ACCT="test-plane-$RANDOM"
TEST_URL="localhost:3030/plane/$TEST_ACCT"

echo "***************************"
echo "* Plane: $TEST_ACCT"
echo "***************************"

echo "Registering plane"
curl --location --request POST $TEST_URL --header 'Content-Type: application/json' --data-raw "{\"Register\": {\"registration_id\": \"$TEST_ACCT\"}}"

echo -e "\nUpdating position"
curl --location --request POST $TEST_URL --header 'Content-Type: application/json' --data-raw "{\"UpdatePosition\": {\"latitude\": 1.0, \"longitude\": 2.0, \"altitude\": 0 }}"

# [...]
```

[rust-cqrs/test/test_api_plane.sh](https://github.com/aurelien-clu/example-ddd-es/blob/main/rust-cqrs/test/test_api_plane.sh)

```bash
# terminal 1
docker-compose up -d
cargo run

# terminal 2
./test/test_api_plane.sh
```

#### **Expected terminal 1 output**

[rust-cqrs/crates/domain-plane/src/queries/logger.rs](https://github.com/aurelien-clu/example-ddd-es/blob/main/rust-cqrs/crates/domain-plane/src/queries/logger.rs) logs every `Plane` events, useful for debugging.

Format:

> id: '{{Aggregate ID}}', sequence: {{Event number for the given aggregate}}
> 
> {{event as a JSON unless the event contains no data (e.g. "OnGround")}}

```
******************************************************
id: 'test-plane-27355', sequence: 1
{
  "Registered": {
    "registration_id": "test-plane-27355"
  }
}
******************************************************
id: 'test-plane-27355', sequence: 2
"OnGround"
******************************************************
id: 'test-plane-27355', sequence: 3
{
  "PositionedAt": {
    "latitude": 1.0,
    "longitude": 2.0,
    "altitude": 0
  }
}
******************************************************
id: 'test-plane-27355', sequence: 4
"TookOff"
******************************************************
id: 'test-plane-27355', sequence: 5
{
  "PositionedAt": {
    "latitude": 10.0,
    "longitude": 20.0,
    "altitude": 10
  }
}
******************************************************
id: 'test-plane-27355', sequence: 6
{
  "PositionedAt": {
    "latitude": 11.0,
    "longitude": 21.0,
    "altitude": 20
  }
}
******************************************************
id: 'test-plane-27355', sequence: 7
"Landed"
```


#### **Expected terminal 2 output**

The journey shows the plane's status and the current or last journey positions (history is cleared upon take off)

```
***************************
* Plane: test-plane-27355
***************************
Registering plane

Updating position

Prepare for take off!

Updating position

Journey
{"registration_id":"test-plane-27355","status":"InAir","positions":[{"latitude":10.0,"longitude":20.0,"altitude":10}]}

Updating position

Prepare for usual landing!

Journey
{"registration_id":"test-plane-27355","status":"OnGround","positions":[{"latitude":10.0,"longitude":20.0,"altitude":10},{"latitude":11.0,"longitude":21.0,"altitude":20}]}
```

## Links

- [wikipedia.org/domain driven design](https://en.wikipedia.org/wiki/Domain-driven_design)
- [martinfowler.com/tags/domain driven design](https://martinfowler.com/tags/domain%20driven%20design.html)
- [event sourcing](https://en.wikipedia.org/wiki/Event-driven_architecture)
- [command query responsibility segregation](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation#Command_Query_Responsibility_Segregation)
- books
  - [Implementing DDD](https://kalele.io/books/)
  - [DDD Distilled](https://kalele.io/books/)
