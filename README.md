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

## Permissions for Workspaces

## Permissions for Channels

## Permissions for Threads and Messages
