# sqerl #

This is the repository for sqerl, a database layer originally written
for chef for interfacing with postgres.

## Getting Started

So, you want to model a Postgres database table with Sqerl. Let's get
to it.

## Postgres I'm going to assume you know very little about Postgres,
since that's what I knew when I started writing this. You can safely
skip this section if you know how to get that running.

You can get Postgres up and running on a Mac as easy as downloading
[Postgres.app](http://postgresapp.com). You should probably learn more
that the basics if you're going to run it anywhere more complex than
that.

Postgres.app starts Postgres on port 5432 of localhost. It'll also
create a user and database for your OS X user's shortname. We'll use
`tyktorp` for the examples below.

We should at least create a table so you have something to interact
with in the following examples.

```sql
CREATE TABLE things (
    id BIGSERIAL PRIMARY KEY,
    description TEXT
);
```

## Add It!

To add sqerl to your project (hereon known as `sample_app`), add it to
your deps proplist in `rebar.config`

```erlang
{sqerl, ".*",
 {git, "git://github.com/opscode/sqerl.git", {branch, "master"}}}
```

It's going to pull in these dependencies, just go with it:

```
Cloning into 'epgsql'...
Cloning into 'pooler'...
Cloning into 'envy'...
```

## Configuring Sqerl

You actually need two `sys.config` application blocks. If only there
were an application out there for abstracing configuration settings in
to logical groupings, instead of imposing erlang's application
behavior as a unit of scope.

```erlang
 {sqerl, [
          %% Database connection parameters
          {db_host, "localhost" },
          {db_port, 5432 },
          {db_user, "tyktorp" },
          {db_pass, "" },
          {db_name, "tyktorp" },
          {idle_check, 10000},
          {column_transforms, []},
          {prepared_statements,
            {sqerl_rec, statements, [[{app, sample_app}]]}}
         ]},

 {pooler, [
           {pools, [[{name, sqerl},
                     {max_count,  10 },
                     {init_count, 5 },
                     {start_mfa, {sqerl_client, start_link, []}}]]}
          ]},
```

## Model the table

`thing.erl`

```erlang
-module(thing).

-compile({parse_transform, sqerl_gobot}).

-export([
         '#insert_fields'/0,
         '#update_fields'/0,
         '#statements'/0,
         '#table_name'/0
        ]).

%% Must be the same as the module name
-record(thing, {
          id :: integer(),
          description :: binary()
         }).

'#insert_fields'() ->
    [description].

'#update_fields'() ->
    [description].

'#statements'() ->
    [default].

%% Only needed if table name is different than module name
%'#table_name'() ->
%    "things".

```

**NOTE:** module name and record name **MUST** be the same.

Hopefully the `sqerl_gobot` parse transform doesn't do too much magic
for you behind the scenes. Just the right amount of magic. They're not
tricks, they're illusions. Here's what it gives you:

### Behavior
`-behaviour(sqerl_rec)` - Basically defines the required callbacks.

### Callbacks

Some callbacks are generated by the parse transform, some you have to
do yourself. Either way, the ones that start with `#` are meant to be
called internally by `sqerl_rec`. The ones that don't are there for
developer use. I suppose you can use them if you want. It's more of a
guideline.

**Generated:**

`'#new'()` - returns a new record of this type

`'is'(Obj)` - is `Obj` an "object" of this type

`getval(Fieldname, Obj)` - returns `Fieldname` of `Object`

`setvals([{Fieldname, Value}|...]=Proplist, Obj)` - returns a copy of
`Obj` with each `Fieldname`'s `Value`' from `PropList` modified.

`fromlist([{Fieldname, Value}|...]=Proplist)` - works just like
`setvals/2`, but it's for new instances only.

`fields()` - returns a list of fields you provided in the `-record` of
your module.

**Write Yourself:**

`'#insert_fields'() -> [atom()]` - the list of record fields to be
inserted with generated insert statement.

`'#update_fields'() -> [atom()]` - the list of record fields to be
updated with generated update statement.

`'#statements'() -> [ default | {atom(), iolist()}]` - a list of named
statements. This is where your specific database code goes.

**Optional:**

`'#table_name'() -> atom`. If your table name isn't the name of your
module, set it here.

## Console

I skipped some steps here for now, but I'm assuming you can open up an
erlang console with this application started.

```erlang
%%% Make sure it's all configured right.
(sample_app@127.0.0.1)> sqerl_rec:statements([thing]).
[{thing_delete_by_id,<<"DELETE FROM things WHERE id = $1">>},
 {thing_fetch_by_id,<<"SELECT id, description FROM things WHERE id = $1">>},
 {thing_insert,<<"INSERT INTO things(description) VALUES ($1) RETURNING id, description">>},
 {thing_update,<<"UPDATE things SET description = $1 WHERE id = $2 RETURNING id, description">>}]

%% Add a thing?
(sample_app@127.0.0.1)> R = {thing, undefined, "Hi!"}.
{thing,undefined,"Hi!"}

(sample_app@127.0.0.1)> sqerl_rec:insert(R).
[{thing,1,<<"Hi!">>}]

(sample_app@127.0.0.1)> sqerl_rec:fetch(thing, id, 1).
[{thing,1,<<"Hi!">>}]

```

And look in the database!

```
tyktorp=# SELECT * FROM things;
 id | description
----+-------------
  1 | Hi!
(1 row)
```

## Here we are now, it's a statement
Let's get everything!

```erlang
(sample_app@127.0.0.1)> sqerl_rec:fetch_all(thing).
{error,{query_not_found,thing_fetch_all}}
%% OOPS!

```

You have to specify a `fetch_all` query for `thing`, fortunately
there's a convenience method for that. Change `thing:#statements/0` to

```
'#statements'() ->
    [default,
        {fetch_all, sqerl_rec:gen_fetch_all(thing, id)}
    ].
```

Yay!

```
(sample_app@127.0.0.1)1> sqerl_rec:fetch_all(thing).
[{thing,1,<<"Hi!">>}]
```

You can make lots of custom statements as your usecase requires. Ours
is pretty trivial though, I probably should have added more columns.


## tests

local postgres commands must be on `$PATH` to successfully `make all`

A set of integration and performance tests can be run via `make ct`

# LINKS:

Source:

    https://github.com/opscode/sqerl

Tickets/Issues:

    http://tickets.opscode.com/

Documentation:

    http://wiki.opscode.com/display/chef/Home/

# LICENSE:

Copyright 2011-2014 Opscode, Inc. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you
may not use this file except in compliance with the License.  You may
obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
implied.  See the License for the specific language governing
permissions and limitations under the License.