Based on the tutorial at [https://hasura.io/learn/graphql/hasura-auth-slack/](https://hasura.io/learn/graphql/hasura-auth-slack/).

# Course Introduction

This course is a super fast introduction to model and think about Auth with Hasura GraphQL.

In 30 mins, you will setup a powerful, realtime and secure GraphQL Backend for Slack Clone.

## Pre-requisites

- You should have some familiarity with Hasura to quickly dive into the Auth sections that this course focuses on. In case you are new to Hasura, we recommend you go over the [Introduction to Hasura Backend Course](https://hasura.io/learn/graphql/hasura/introduction/) before taking this.

If you are just getting started with Hasura or would like a refresher, please visit [https://github.com/TheRobBrennan/explore-hasura-basics](https://github.com/TheRobBrennan/explore-hasura-basics).

## What will I learn?

This course will help you understand how to think about permissions and access control with Hasura.

- Roles: Define roles based on business requirements
- Access Control: Who can access what part of the database.
- Authorization Mode: Setup authorization so that app users can only run operations on data that they should be allowed to.
- Authentication: Integrate a JWT based auth provider (Node.js/Passport) with Hasura.
- Auth with external services: Add a custom GraphQL resolver and forward headers to handle permissions.
- Allow Lists: Go production ready by allowing only a list of queries you specify.
- Client side implementations: How to setup headers in simple http requests, web sockets for realtime data and custom x-hasura-\* headers.

## What will we be building?

We will be building the backend of a basic Slack clone, setting up permissions so that the right data is exposed to the right user. There won't be any frontend app building associated with this tutorial.

## What do I need to take this tutorial?

- Hasura CLI (Docs)
- Node.js 8+ installed to setup the Auth Server later.

We've kept this course light on developer workflows and environment choices so that you can focus on the key concepts and go on to set up your favorite tools and workflows.

## How long will this tutorial take?

Around 45 mins.

# Deploy Hasura

Let's start by deploying Hasura.

## One-click deployment on Heroku

The fastest way to try Hasura out is via Heroku.

- Click [https://heroku.com/deploy?template=https://github.com/hasura/graphql-engine-heroku](https://heroku.com/deploy?template=https://github.com/hasura/graphql-engine-heroku) to deploy GraphQL Engine on Heroku with the free Postgres add-on.

This will deploy Hasura GraphQL Engine on Heroku. A PostgreSQL database will be automatically provisioned along with Hasura. If you don’t have an account on Heroku, you would be required to sign up. Note: It is free to signup and no credit-card is required.

Type in the app name - `rb-explore-hasura-auth-slack` in my example - and select the region of choice and click on `Deploy app`.

## Hasura Console

Once the app is deployed, click on the View button to open the app.

For this demo, you can view [https://rb-explore-hasura-auth-slack.herokuapp.com](https://rb-explore-hasura-auth-slack.herokuapp.com)

Great! You have now deployed Hasura GraphQL Engine and have the admin console ready to get started!

# Data Modeling: Slack

In this part of the course, we will build the data model for a realtime slack clone. Our slack app will have the following basic features:

- Users can signup.
- Users can create workspaces.
- Workspaces can be managed by the owner of the workspace or the admin of the workspace.
- Users can be added to channels in the workspace they are part of.
- Users can send messages to channels in the workspace they are part of.
- Users can send messages to other users in the same workspace.
- Users can see which users are online in the workspace they are part of.
- Broadly this means that we have few top level models in this app.

We will go over them in the subsequent steps.

## Tables for Slack Clone

Let's get started by looking at the data model.

### Users

The primary functionality of the app revolves around users and their messages.

So we have the following tables.

- `users` and `user_message`

### Workspace

Slack app has workspaces where users can join. It is managed by the owner and the admins of the workspace. The following tables takes care of this requirement.

- `workspace`, `workspace_member` and `workspace_user_type`

Each workspace can have channels scoped to a specific topic of discussion having subset of members from the workspace. Members of the channel can post messages to the channel that everyone can see.

### Channel

- `channel`, `channel_member`, `channel_thread` and `channel_thread_message`

The final model roughly looks like the following with basic relational columns:

![https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/slack-datamodel.png](https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/slack-datamodel.png)

Note that it doesn't have the detailed column list, but it should give an idea of the relationships between different entities.

## Relationships

Relationships enable you to make nested object queries if the tables/views in your database are connected.

GraphQL schema relationships can be either of

- object relationships (one-to-one)
- array relationships (one-to-many)

### Object Relationships

Let's say you want to query `workspace` and more information about the `user` who created it. This is achievable using nested queries if a relationship exists between the two. This is a one-to-one query and hence called an object relationship.

An example of such a nested query looks like this:

```gql
query {
  workspace {
    id
    name
    owner {
      id
      name
    }
  }
}
```

In a single query, you are able to fetch workspace and its related user information. This can be very powerful because you can nest to any level.

### Array Relationships

Let's look at an example query for array relationships:

```gql
query {
  users {
    id
    name
    messages {
      id
      message
      channel_id
    }
  }
}
```

In this query, you are able to fetch users and for each user, you are fetching the messages (multiple) sent by that user. Since a user can have multiple messages, this would be an array relationship.

Relationships can be captured by foreign key constraints. Foreign key constraints ensure that there are no dangling data. Hasura Console automatically suggests relationships based on these constraints.

Though the constraints are optional, it is recommended to enforce these constraints for data consistency.

The above queries won't work yet because we haven't defined the relationships yet. But this gives an idea of how nested queries work.

## Apply Migrations

Let's get started by creating the tables and relationships for the Slack app.

Download the hasura project with migrations from [here](https://hasura.io/learn/graphql/hasura-auth-slack/slack-backend.zip)

First, let's make sure that we have installed our dependencies for this project:

```sh
$ npm install
```

Configure the endpoint to point to the heroku app URL. Open the `slack-backend/config.yaml` file and set the endpoint value:

```yml
endpoint: https://rb-explore-hasura-auth-slack.herokuapp.com
```

Now let's apply the migrations:

```sh
$ cd slack-backend/
$ npx hasura migrate apply
```

This will create the tables and relationships for the slack app.

Great! Now navigate to the heroku app - [https://rb-explore-hasura-auth-slack.herokuapp.com](https://rb-explore-hasura-auth-slack.herokuapp.com) for my example - to see the tables with relationships.

## Try out GraphQL APIs

As you are aware that Hasura gives you Instant GraphQL APIs over Postgres, it can be tested on the table that we just created.

Let's go ahead and start exploring the GraphQL APIs for `users` table.

### Mutation

Head over to Console -> GRAPHIQL tab and insert a user using GraphQL Mutations:

```gql
mutation {
  insert_users(
    objects: [
      { name: "Praveen", email: "myemail@example.com", password: "password123" }
    ]
  ) {
    affected_rows
  }
}
```

Click on the `Play` button on the GraphiQL interface to execute the query.

You should get a response looking something like this:

```js
{
  "data": {
    "insert_users": {
      "affected_rows": 1
    }
  }
}
```

Great! You have now consumed the mutation query for the `users` table that you just created. Easy isn't it?

Tip: You can use the `Explorer` on the GraphiQL interface to generate the mutation in a few clicks.

### Query

Now let's go ahead and query the data that we just inserted:

```gql
query {
  users {
    id
    name
    created_at
  }
}
```

You should get a response looking something like this:

```js
{
  "data": {
    "users": [
      {
        "id": "686b21d3-67e3-4285-bae8-9994f32a646e",
        "name": "Praveen",
        "created_at": "2020-06-20T04:34:37.632368+00:00"
      }
    ]
  }
}
```

Note that some columns like `created_at` have default values, even though you did not insert them during the mutation.

### Subscription

Let's run a simple subscription query over `users` table to watch for changes to the table:

```gql
subscription {
  users {
    id
    name
    created_at
  }
}
```

Initially the subscription query will return the existing results in the response.

Now let's insert new data into the users table and see the changes appearing in the response.

In a new tab, Head over to Console -> `DATA` tab -> users -> Insert Row and insert another row.

And switch to the previous `GRAPHIQL` tab and see the subscription response returning 2 results.

An active subscription query will keep returning the latest set of results depending on the query.

# Thinking in Roles

In this part of the tutorial, we will look at how to model roles for the app.

Role based access control lets the server control what data is accessed by each user on the client. This can enforce granular restrictions on data access.

Let's think about the different set of roles applicable to users of Slack.

We can broadly classify roles as:

- Hierarchical and Flat or
- Administrative and Non-Administrative

Every member in Slack has a role and each one has a different level of permissions. For example, every workspace in Slack has an owner who created it. The owner, along with a few admins would be able to completely manage the workspace where as the members of the workspace just get to participate.

On top of all these there's an admin role who can do everything in the backend from creating workspaces, users and deleting records.

Let's dissect each data model to see who can do what.

## Defining Roles

In this section, we will look at how to generally approach modelling roles for permissions with Hasura.

Traditionally you will consider multiple roles based on responsibilities assigned to user.

In the slack app model, a quick analysis will give you possibilities for multiple roles. There are `users` of the app, `workspace owners` who control and manage the workspace, `workspace admins` who have a subset of permissions to manage the workspace, `channel admins` and so on. But the important thing to note is, they are all `users` of the app with just extra permissions to read / write some data. This leaves us to define just a single role called user which can accommodate the above permission layer. We will see how in the sections later.

You will most likely need only one role with Hasura for users of your app. But there are cases where you genuinely need multiple roles to control data access.

### The case for multiple roles

So when are multiple roles used for defining permissions? Let's take a look at the some of the use cases.

#### Logged in vs Publicly accessible data

When some part of the data in the app is publicly visible but some are available only for logged in users, then multiple roles is the way forward. In the slack app, everything is assumed to be available only for logged in users.

#### Different access to columns

In cases where the columns which can be accessed differ based on who is logged in, then multiple roles are used. For example, in the slack app model, the workspace owner can see some columns which are sensitive and restricted read access to other users, then naturally we need to define multiple roles.

#### Backend support / admin teams

If your app has admin/customer support/analytics teams which needs read access across tables without restrictions, then they will have their own individual roles.

You can get away with a single role if you don't have the above constraints.

## User role for the app

In this realtime slack app, we need to restrict all querying only for logged in users. We assume that data is not publicly accessible. Everything revolves around what users do on the app. Also certain columns in tables need not be exposed to the user.

Let's see the different responsibilities that a user can have.

### Administrative

All administrative tasks require write access to the database. Some of the administrative tasks are

- Create and manage workspaces
- Create and manage channels
- Add members to workspace and channel

### Non Administrative

Non-administrative tasks require scoped read and write access to the database.

For example, in a Slack app you have Members. They can join a Slack workspace. They can use Slack to communicate and collaborate with other members.

- User can read and send messages to channels
- User can read and send messages to other users in the same workspace

We need to be able to apply these actions to a role. We will see how in the next section.

# Access Control

In this part of the tutorial, we are going to define role based access control rules for each of the models that we created. Access control rules help in restricting querying on a table based on certain conditions.

Access control rules can be applied on

- Row level
- Column level

## Row Level

With row level access control, users can access tables without having access to all rows on that table. This is particularly useful to protect sensitive personal data which is part of the table. This way, you can allow all users to access a table, but only a specific number of rows in that table.

![https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/row-level-access-control.png](https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/row-level-access-control.png)

## Column Level

Column level access control lets you restrict access to certain columns in the table. This is useful to hide data which are not relevant, sensitive or used for internal purposes. A typical representation of data looks like:

![https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/column-level-access-control.png](https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/column-level-access-control.png)

As you can imagine, combining both these rules gives a flexible and powerful way to control data access to different stakeholders involved.

## Types of operation

Access control rules can be applied to all the CRUD operations (Create, Read, Update and Delete). Some operations can be completely restricted to not allow the user perform the operation.

In the previous section we learnt that the slack app requires a role called `user`. We will create permissions for this role in the next part.

## Permissions for Users

Slack app revolves around users. We start off by setting permission rules for the users of the app for the CRUD operations.

### Select permission

What user data is allowed to be read by a logged in user of Slack?

All logged in users can read user data of those who belong to the same workspace as the logged in user.

The requirement translates into something like:

- You can read your own user data.
- You can read user data of others who are part of the same workspace that you are a member of.

This is a typical boolean expression where you say the one who is trying to access a record in the users table must either belong to them `id = X-Hasura-User-Id` or they must be part of the same workspace `workspace_members.user_id = X-Hasura-User-Id`

#### Row level select

The expanded valid boolean expression of the above statement looks like this:

```js
{
  "_or": [
    {
      "id": {
        "_eq": "X-Hasura-User-Id"
      }
    },
    {
      "workspace_members": {
        "user_id": {
          "_eq": "X-Hasura-User-Id"
        }
      }
    }
  ]
}
```

#### Column level select

After filtering out the rows that a user is supposed to acccess, we need to filter out which fields they are allowed to read. Apart from the `password` field, every column in the users table is accessible by any authenticated user, since there is no sensitive data that needs to be restricted to only the user.

![https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/slack-users-select-columns.png](https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/slack-users-select-columns.png)

We are done with read access. Let's move on to write access which lets a user to either create, update or delete.

### Insert permission

Are the users of the app allowed to directly insert into `users` table? No. A user signs up on the app which goes through the auth server which deals with user registration, validation, triggering welcome email and so on. Hence the auth server with access to admin role will insert the record into the `users` table post validation and generating the right token. We can skip defining permisisons for the user role's insert operation.

### Update permission

Who is allowed to update the existing data in `users` table?

As an authenticated user of the app, i should be able to update ONLY my own personal data.

#### Row level update

The above condition translates to the following expression:

```js
{
  "id": {
    "_eq": "X-Hasura-User-Id"
  }
}
```

Update the row only if the `id` of the column matches the id value of the authenticated user (`X-Hasura-User-Id`)

#### Column level update

We need to fix up on what columns the user is allowed to update directly from the app. A simple checklist would be to not allow the user to update their own `id`, `email`, and `created_at`. We are also going to restrict them to directly modify the `password` column since that is delegated to the auth server which does the necessary validation before the update.

### Delete permission

We do not want to allow the user to delete their own user record directly from the app and hence we can skip defining rules for this operation. This is assumed to be done by Auth servers which handles user management post validations for deleting user accounts.

### Potential for other roles

All of the above rules were applied for the user role. But let's say there are fields which are private to the user and not meant to be read by other users. For example, in our current model, `phone_number` field is assumed to be public. In case the requirement for it is to be private to the user, then we need to create a new role, let's call it `me` and define rules for select permission without the `phone_number` column.

The row level select rule will translate into something like this:

```js
{
  "id": {
    "_eq": "X-Hasura-User-Id"
  }
}
```

And the column level permission remains the same for the role `me` but the role `user` will not have access to `phone_number` column.

## Permissions for Workspaces

### Select permission

Which workspace data is allowed to be read by a logged in user of Slack?

- Anybody who is a member of a workspace should be able to read data about their workspace.

This is a typical boolean expression where you say the one who is trying to access a record in the workspace table must either be the owner `owner_id = X-Hasura-User-Id` or they must be part of the same workspace `workspace_members.user_id = X-Hasura-User-Id`

#### Row level select

The expanded valid boolean expression of the above statement looks like this:

```js
{
  "_or": [
    {
      "owner_id": {
        "_eq": "X-Hasura-User-Id"
      }
    },
    {
      "workspace_members": {
        "user_id": {
          "_eq": "X-Hasura-User-Id"
        }
      }
    }
  ]
}
```

#### Column level select

After filtering out the rows that a user is supposed to acccess, we need to filter out which fields they are allowed to read. Since there is no sensitive data that needs to be restricted to only a certain type of user, we give permission to select for all columns.

We are done with read access. Let's move on to write access which lets a user to either create, update or delete.

### Insert permission

Are the users of the app allowed to directly insert into `workspace` table? Yes, any authenticated user is allowed to create a workspace on their own. It translates into the following expression:

```js
{
  "owner_id": {
    "_eq": "X-Hasura-User-Id"
  }
}
```

#### Column presets

You can set static values or session variables as default values for the column while doing an insert.

In the workspace table, the owner_id should be automatically set to the session variable `X-Hasura-User-Id` and the user shouldn't be allowed to set this value. We use Column Presets in this case to achieve this.

![https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/slack-workspace-user-insert.png](https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/slack-workspace-user-insert.png)

### Update permission

Who is allowed to update the existing data in `workspace` table?

Only an authenticated user of the app and the owner of the workspace should be allowed to update the data in the workspace.

#### Row level update

The above condition translates to the following expression:

```js
{
  "owner_id": {
    "_eq": "X-Hasura-User-Id"
  }
}
```

Update the row only if the `owner_id` of the column matches the id value of the authenticated user (`X-Hasura-User-Id`)

#### Column level update

We need to fix up on what columns the user is allowed to update directly from the app. A simple checklist would be to NOT allow the user to update the `id`, `owner_id`, and `created_at` values. The remaining columns can be allowed.

### Delete permission

The owner of the workspace should be the only user who should be able to delete the workspace. This again translates into the following expression:

```js
{
  "owner_id": {
    "_eq": "X-Hasura-User-Id"
  }
}
```

Do note that, in case the workspace is deleted all the dependent records in all other tables should also be removed. Hence this can be done as a single operation by the admin role at the server side instead of allowing direct delete of the workspace from the client. The other option is to use ON DELETE triggers to perform a cascade delete which will remove all the dependent rows across the database.

## Permissions for Channels

### Select permission

We need to see what channel data is accessible to users. The criteria looks simple:

- Anybody who is a member of a channel should be able to read data about that channel.

This is a typical boolean expression where you say the one who is trying to access a record in the channel table must be part of the channel `channel_members.user_id = X-Hasura-User-Id`

#### Row level select

The expanded valid boolean expression of the above statement looks like this:

```js
{
  "channel_members": {
    "user_id": {
      "_eq": "X-Hasura-User-Id"
    }
  }
}
```

#### Column level select

After filtering out the rows that a user is supposed to acccess, we need to filter out which fields they are allowed to read. Since there is no sensitive data that needs to be restricted to only a certain type of user, we give permission to select for all columns.

We are done with read access. Let's move on to write access which lets a user to either create, update or delete a channel.

### Insert permission

Are the users of the app allowed to directly insert into `channel` table? Yes, any authenticated user who is an owner or an admin is allowed to create a channel on their own. But they can create channels only in workspaces they are part of. It translates into the following expression:

```js
{
  "_and": [
    {
      "workspace": {
        "workspace_members": {
          "user_id": {
            "_eq": "X-Hasura-User-Id"
          }
        }
      }
    },
    {
      "workspace": {
        "workspace_members": {
          "user_type": {
            "type": {
              "_in": [
                "owner",
                "admin"
              ]
            }
          }
        }
      }
    }
  ]
}
```

We use the `_and` boolean expression to say both conditions need to be satisfied. The user_type table is an enum with values `owner`, `admin` and `member`. Both owners and admins of the workspace can create a channel and hence the above expression.

#### Column Presets

In the channel table, the `created_by` should be automatically set to the session variable `X-Hasura-User-Id` and the user shouldn't be allowed to set this value. We use Column Presets in this case to achieve this.

### Update permission

Who is allowed to update the existing data in `channel` table?

Only an authenticated user of the app and the owner or admin of the workspace should be allowed to update the data in the workspace.

#### Row level update

The above condition translates to the following expression (same as above):

```js
{
  "_and": [
    {
      "workspace": {
        "workspace_members": {
          "user_id": {
            "_eq": "X-Hasura-User-Id"
          }
        }
      }
    },
    {
      "workspace": {
        "workspace_members": {
          "user_type": {
            "type": {
              "_in": [
                "owner",
                "admin"
              ]
            }
          }
        }
      }
    }
  ]
}
```

#### Column level update

We need to fix up on what columns the user is allowed to update directly from the app. Only a channel's public status and name can be allowed to change. (`is_public` and `name`).

### Delete permission

The owner of the workspace and the admins should be the only user(s) who should be able to delete the channel. This again translates into the above boolean expression (same as insert and update).

Do note that, in case the channel is deleted all the dependent records in all other tables should also be removed. Hence this can be done as a single operation by the admin role at the server side instead of allowing direct delete of the channel from the client. The other option is to use ON DELETE triggers to perform a cascade delete which will remove all the dependent rows across the database.

## Permissions for Threads and Messages

We are done with rules for all the base tables (`users`, `workspace` and `channel`). The primary part of slack is for users to send and receive messages on the channel or to other users. Let's see how that is applicable in the access control rules.

Let's start with the `channel_thread` and `channel_thread_message` tables.

### Select permission

We need to list down who can access a message posted on any channel. The requirement looks like:

- Anybody who is a channel member should be able to access all channel threads.

#### Row level select

The expression for `channel_thread` roughly translates to the following:

```js
{
  "channel": {
    "channel_members": {
      "user_id": {
        "_eq": "X-Hasura-User-Id"
      }
    }
  }
}
```

The expression differs slighlty for `channel_thread_message` since it has one more level of nesting:

```js
{
  "channel_thread": {
    "channel": {
      "channel_members": {
        "user_id": {
          "_eq": "X-Hasura-User-Id"
        }
      }
    }
  }
}
```

#### Column level select

After filtering out the rows that a user is supposed to acccess, we need to filter out which fields they are allowed to read. Since there is no sensitive data that needs to be restricted to only a certain type of user, we give permission to select for ALL columns.

We are done with read access. Let's move on to write access which lets a user to either create, update or delete a channel.

### Insert permission

Any authenticated user who is a part of a workspace can post messages on the channels of the workspace. It translates into the same expression as above for `channel_thread` table.

### Update permission

Users are not allowed to update a `channel_thread`. So then who is allowed to update the existing messages in `channel_thread_message` table?

- Any authenticated user can update their own message posted on any channel.

#### Row level update

The above condition translates to the following expression:

```js
{
  "user_id": {
    "_eq": "X-Hasura-User-Id"
  }
}
```

#### Column level update

The user can only update the message column in `channel_message` table.

### Delete permission

The user who created the message can delete their own message. It translates to the same expression that we defined for the update operation.

Again as in the previous steps, CASCADE delete can be applied to remove all the dependent and dangling data.

# Authentication Modes

In this part, we will look at the different modes for Authentication. Authentication is handled outside of Hasura. You can bring in your own Auth server and integrate it with Hasura. There are broadly two options available.

- JWT Mode
- Webhook Mode

## JWT Mode

You can configure GraphQL Engine to use the JWT authorization mode to authorize all incoming requests. The auth server is expected to return a valid JWT token, which are decoded and verified by the GraphQL engine, to authorize and get metadata about the request.

A typical architecture with Auth server issuing JWT looks like the one below:

![https://hasura.io/docs/1.0/_images/jwt-auth1.png](https://hasura.io/docs/1.0/_images/jwt-auth1.png)

The Auth Server issues JWT tokens with relevant x-hasura-\* claims to the app which then sends the token to Hasura GraphQL Engine. Hasura then validates the claims to allow the request to go through.

## Webhook Mode

You can also configure GraphQL Engine to use the Webhook mode. Your auth server exposes a webhook that is used to authenticate all incoming requests to the Hasura GraphQL engine server and to get metadata about the request to evaluate access control rules.

The architecture with webhook looks like the one below:

![https://hasura.io/docs/1.0/_images/webhook-auth1.png](https://hasura.io/docs/1.0/_images/webhook-auth1.png)

### Unauthenticated Mode

Sometimes you would like to allow access to data without a user being logged in. This is useful for public feed which is open to all users. Although our Slack app doesn't have this as a use case, it is good to know when this could be used.

## Choosing the right mode

In this part, we will look at which mode is right for the Slack clone.

### Using JWT Mode

JWT mode is a recommended solution with Hasura, if your Auth server can support it.

Our Slack app clone doesn't need to integrate with legacy auth systems or has complex custom rules which can only be processed via a webhook. The Auth server can be configured to inject the right hasura claims into every token it generates ensuring that the permission rules can be applied.

The Slack app has users that needs to be assigned roles. JWT mode is the recommended mode due to ease of integration and the benefits it brings for clients.

### When to use webhook mode?

Webhook mode is generally required if the Auth server you use cannot issue JWT tokens in the format that Hasura expects it to be or doesn't have JWT integration at all to begin with. This is a more common use case with existing legacy auth systems. Do note that with a webhook mode, the webhook has to be deployed, maintained and everytime a request is made to Hasura, it will in turn make a request to the webhook to authorize the request. This could have a slight latency depending on where the webhook is deployed.

## Configuring JWT Secret

In this part, we will look at how to configure the JWT secret.

Follow the instructions [here](https://github.com/hasura/learn-graphql/tree/master/services/backend/auth-server) to setup the Auth server.

### Authenticate JWT using GraphQL Engine

The GraphQL engine comes with built in JWT authentication. You will need to start the engine with the same secret/key as the JWT auth server using the environment variable `HASURA_GRAPHQL_JWT_SECRET` (`HASURA_GRAPHQL_ADMIN_SECRET` is also required to set this up). Read more in [docs](https://hasura.io/docs/1.0/graphql/manual/auth/authentication/jwt.html#running-with-jwt)

A sample CURL command using the above token would be:

```sh
curl -X POST \
  https://hasura-auth.herokuapp.com/v1/graphql \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxIiwibmFtZSI6InRlc3QxMjMiLCJpYXQiOjE1NDAzNzY4MTUuODUzLCJodHRwczovL2hhc3VyYS5pby9qd3QvY2xhaW1zIjp7IngtaGFzdXJhLWFsbG93ZWQtcm9sZXMiOlsiZWRpdG9yIiwidXNlciIsIm1vZCJdLCJ4LWhhc3VyYS11c2VyLWlkIjoiMSIsIngtaGFzdXJhLWRlZmF1bHQtcm9sZSI6InVzZXIiLCJ4LWhhc3VyYS1yb2xlIjoidXNlciJ9fQ.w9uj0FtesZOFUnwYT2KOWHr6IKWsDRuOC9G2GakBgMI' \
  -H 'Content-Type: application/json' \
  -d '{ "query": "{ users { id } }" }'
```

Now you can test this out by navigating to console and making queries without the admin secret. You should ideally get an error.

# Production Ready Auth

The Hasura GraphQL API exposes a number of queries to both the admins and regular users of the app. The permissions are clearly defined for each role. But on top of these, you can exactly specify a list of queries that should be executed.

The Allow-list is a list of safe queries (GraphQL queries, mutations or subscriptions) that is stored by the GraphQL engine in its metadata.

You can enable Allow Lists via environment variable called `HASURA_GRAPHQL_ENABLE_ALLOWLIST`.

![https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/allow-list-env-config.png](https://graphql-engine-cdn.hasura.io/learn-hasura/assets/graphql-hasura-auth/allow-list-env-config.png)

In the Slack app we have a number of queries and mutations that can be listed down and only those can be allowed to be executed by the server.

For example, some of the queries required for the slack app are

- Fetch the list of workspaces a user is part of:

```gql
query {
  users {
    workspaces {
      id
      name
    }
  }
}
```

- Fetch the list of channels in a workspace

```gql
query getChannelsInWorkspace($workspaceId: uuid_comparison_exp) {
  channel(where: { workspace_id: $workspaceId }) {
    id
    name
    created_by
  }
}
```

Note that this uses variables and hence the same query with different values for variables will be allowed.

- Fetch the list of messages posted in a channel

```gql
query getChannelsInWorkspace($workspaceId: uuid_comparison_exp, $offset: Int!) {
  channel(where: { workspace_id: $workspaceId }, limit: 20, offset: $offset) {
    id
    name
    channel_threads {
      channel_thread_messages {
        id
        message
      }
    }
  }
}
```
