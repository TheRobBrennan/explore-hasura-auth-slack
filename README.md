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

## Apply Migrations

## Try out GraphQL APIs
