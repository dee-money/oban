# Oban

Logo here
Brief description here

[![Hex Version](https://img.shields.io/hexpm/v/oban.svg)](https://hex.pm/packages/oban)
[![Hex Docs](http://img.shields.io/badge/hex.pm-docs-green.svg?style=flat)](https://hexdocs.pm/oban)
[![CircleCI](https://circleci.com/gh/sorentwo/oban.svg?style=svg)](https://circleci.com/gh/sorentwo/oban)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

# Table of Contents

- Introduction
- Features
  - Isolated queues
  - Scheduled jobs
  - Telemetry integration
  - Reliable execution/orphan rescue
  - Consumer draining, slow jobs are allowed to finish before shutdown
  - Historic Metrics
  - Node Metrics
  - Property Tested
- Why? | Philosophy | Rationale
- [Installation](#Installation)
- [Usage](#Usage)
  - [Configuring Queues](#Configuring-Queues)
  - [Creating Workers](#Creating-Workers)
  - [Enqueuing Jobs](#Enqueueing-Jobs)
- [Testing](#Testing)
- [Contributing](#Contributing)
- [License](#License)

## Installation

Oban is published on [Hex](https://hex.pm/oban). Add it to your list of
dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:oban, "~> 0.2"}
  ]
end
```

Then run `mix deps.get` to install Oban and its dependencies, including
[Ecto][ecto], [Jason][jason] and [Postgrex][postgrex].

After the packages are installed you must create a database migration to
add the `oban_jobs` table to your database:

```bash
mix ecto.gen.migration add_oban_jobs_table
```

Open the generated migration in your editor and delegate the `up` and `down`
functions to `Oban.Migrations`:

```elixir
defmodule MyApp.Repo.Migrations.AddObanJobsTable do
  use Ecto.Migration

  defdelegate up, to: Oban.Migrations
  defdelegate down, to: Oban.Migrations
end
```

Finally, run the migration to create the table:

```bash
mix ecto.migrate
```

Next see [Usage](#Usage) for how to integrate Oban into your application and
start defining jobs!

[ecto]: https::/hex.pm/ecto
[jason]: https::/hex.pm/jason
[postgrex]: https::/hex.pm/postgrex

## Usage

Oban isn't an application, it is started by a supervisor that must be included in your
application's supervision tree.  All of the configuration may be passed into the `Oban`
supervisor, allowing you to configure Oban like the rest of your application.

```elixir
# confg/config.exs
config :my_app, Oban,
  repo: MyApp.Repo,
  queues: [default: 10, events: 50, media: 20]

# lib/my_app/application.ex
defmodule MyApp.Application do
  @moduledoc false

  use Application

  alias MyApp.{Endpoint, Repo}

  def start(_type, _args) do
    children = [
      Repo,
      Endpoint,
      {Oban, Application.get_env(:my_app, Oban)}
    ]

    Supervisor.start_link(children, strategy: :one_for_one, name: MyApp.Supervisor)
  end
end
```

#### Configuring Queues

Queues are specified as a keyword list where the key is the name of the queue
and the value is the maximum number of concurrent jobs. The following
configuration would start four queues with concurrency ranging from 5 to 50:

```elixir
queues: [default: 10, mailers: 20, events: 50, media: 5]
```

There isn't a limit to the number of queues or how many jobs may execute
concurrently. Here are a few caveats and guidelines:

* Each queue will run as many jobs as possible concurrently, up to the
  configured limit. Make sure your system has enough resources (i.e. database
  connections) to handle the concurrent load.
* Only jobs in the configured queues will execute. Jobs in any other queue
  will stay in the database untouched.
* Be careful how many concurrent jobs are called outside the BEAM. The BEAM
  ensures that the system stays responsive under load, but those guarantees
  don't apply when using ports or shelling out.

#### Creating Workers

Worker modules do the work of processing a job. At a minimum they must define a
`perform/1` function, which is called with an `args` map.

Define a worker to process jobs in the `events` queue:

```elixir
defmodule MyApp.Workers.Business do
  use Oban.Worker, queue: "events", max_attempts: 10

  @impl Oban.Worker
  def perform(%{"id" => id}) do
    model = MyApp.Repo.get(MyApp.Business.Man, id)

    IO.inspect(model)
  end
end
```

The return value of `perform/1` doesn't matter and is entirely ignored. If the
job raises an exception or throws an exit then the error will be reported and
the job will be retried (provided there are attempts remaining).

#### Enqueueing Jobs

Jobs are simply Ecto strucs and are enqueued by inserting them into the
database. Here we insert a job into the `default` queue and specify the worker
by module name:

```elixir
%{id: 1, user_id: 2}
|> Oban.Job.new(queue: :default, worker: MyApp.Worker)
|> MyApp.Repo.insert()
```

For convenience and consistency all workers implement a `new/2` function that
converts an args map into a job changeset suitable for inserting into the
database:

```elixir
%{in_the: "business", of_doing: "business"}
|> MyApp.Workers.Business.new()
|> MyApp.Repo.insert()
```

The worker's defaults may be overridden by passing options:

```elixir
%{vote_for: "none of the above"}
|> MyApp.Workers.Business.new(queue: "special", max_attempts: 5)
|> MyApp.Repo.insert()
```

Jobs may be scheduled down to the second any time in the future:

```elixir
%{id: 1}
|> MyApp.Workers.Business.new(schedule_in: 5)
|> MyApp.Repo.insert()
```

## Testing

Oban doesn't provide any special mechanisms for testing. However, here are a few
recommendations for running tests in isolation.

* Set a high `poll_interval` in your test configuration. This effectively stops
  queues from polling and will prevent inserted jobs from executing.

  ```elixir
  config :my_app, Oban, poll_interval: :timer.minutes(30)
  ```

* Be sure to use the Ecto Sandbox for testing. Oban makes use of database pubsub
  events to dispatch jobs, but pubsub events never fire within a transaction.
  Since sandbox tests run within a transaction no events will fire and jobs
  won't be dispatched.

  ```elixir
  config :my_app, Oban, pool: Ecto.Adapters.SQL.Sandbox
  ```

## Contributing

To run the Oban test suite you must have PostgreSQL 10+ running locally with a
database named `oban_test`. Follow these steps to create the database, create
the database and run all migrations:

```bash
# Create the database
MIX_ENV=test mix ecto.create -r Oban.Test.Repo

# Run the base migration
MIX_ENV=test mix ecto.migrate -r Oban.Test.Repo
```

To ensure a commit passes CI you should run `mix ci` locally, which executes the
following commands:

* Check formatting (`mix format --check-formatted`)
* Lint with Credo (`mix credo --strict`)
* Run all tests (`mix test`)
* Run Dialyzer (`mix dialyzer --halt-exit-status`)

## License

Oban is released under the MIT license. See the [LICENSE](LICENSE.txt).
