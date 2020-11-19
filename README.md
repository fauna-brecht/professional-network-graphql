# About this repository
This project is an example application that was built specifically to do GraphQL demos and is definitely not a complete application yet. We aim to finish it and write complete example applications with GraphQL in the near future just like we did with the Fauna Query Language [here](https://css-tricks.com/rethinking-twitter-as-a-serverless-app/)
In the meantime we'll release this simple demo since many users asked about it. 

This repository was part of a workshop which you can watch here: https://www.youtube.com/watch?v=_kEDBitNbnY and which can be followed by running through this document: https://docs.google.com/document/d/10HrtNsaQH0MBTsRKJIQldEr9XMiTLWkEsJ0HYEIrmSU/edit where the answers are provided in white text. 

# Disclaimer
Since the repository is used as a workshop repository, the default branch **does not contain** all the code. To set up the project with all code included you will need to go to the final branch:

```
git checkout final
```

## What it shows
This app shows how to set up a GraphQL endpoint in minutes and further goes into authentication with FaunaDB without a backend and implements some simple queries with pagination. 

## Setup the project

### Create a database
Log in to the FaunaDB [dashboard](https://dashboard.fauna.com/) and create a new database by clicking on *New Database*
Give it a name and click *Save*

### Import the schema
Setting up a GraphQL endpoint in FaunaDB is all about importing the schema which you can find in this repository under src/
The schema looks a follows: 

```
type Account {
  email: String! @unique
}

type Profile {
  name: String!
  icon: String!
  account: Account! @relation
  skills: [ Skill ! ] @relation
  projects: [ Project! ] @relation
}

type Project {
  name: String!
  profile: [ Profile! ] @relation
}

type Skill {
  profiles: [ Profile! ] @relation
  name: String!
}

type Query {
  allProfiles: [Profile!]
  accountsByEmail(email: String!): [Account!]!
  skillsByName(name: String!): [Skill!]!
}

type Mutation {
  register(email: String!, password: String!): Account! @resolver
  login(email: String!, password: String!): String! @resolver
}
```

Go to the GraphQL tab in the FaunaDB [dashboard](https://dashboard.fauna.com/) and click import schema and select the schema.gql file. 
You now have a GraphQL endpoint and should get a playground to play around with it. 

### Run the setup scripts

We have added scripts to set up all the security roles, collections, indexes to make this work. 
The scripts are meant to get you started easily and to document the process. Take a peek in the scripts/setup.js script to see
how this is setup. To run the script, create an Admin token on https://dashboard.fauna.com/ via the Security tab, copy the token
and run: 

`yarn run setup`

The script will ask you for the token, paste it in when it does. 

### Update the UDFs
When you use a @resolver, it will create a UDF stub for you but since a resolver is custom FQL code, that FQL can't be generated for you.
Go to the Fauna dashboard again, click on Functions and then edit the code for the login function:

```
Query(Lambda(['email', 'password'],
    Select(
      ['secret'],
      Login(Match(Index('accountsByEmail'), Var('email')), {
        password: Var('password')
      })
    )
))
```

And the register function:

```
Query(Lambda(['email', 'password'],
    Create(Collection('Account'), {
      credentials: { password: Var('password') },
      data: {
        email: Var('email')
      }
  })
))
```

The script will ask for the admin token, do not use the admin token for anything else than the setup script. 
Admin tokens are powerful and meant to manipulate all aspects of the database (create/drop collections/indexes/roles)
The script will give you a new token instead (a login token).
Copy the token and place it in a .env.local file:
`
REACT_APP_BOOTSTRAP_FAUNADB_KEY=<YOUR FAUNA LOGIN KEY>
`

Note: make sure not to create the UDF before you upload the GraphQL schema, it has to be created with the GraphQL schema since this will include metadata for the UDF to set it up that it can be called from GraphQL.

## Run the project
This project has been created with create-react-app and therefore has all the same commands such as 
`yarn start`

# Security notes:
FaunaDB's security roles are extremely flexible, in combination with User Defined Function we can 
bootstrap the security. We start off with a token that can only call two User Defined Functions (like stored procedures) functions (register, login).
Once the user logs in, the token is swapped with a 'login token' which has access to view profiles. 

## What it does not do

### Short-lived tokens
At this point, FaunaDB does not provide short-lived tokens but it **can** be implemented fairly easily since FaunaDB tokens are accessible with an admin token. That means you could write a serverless function that periodically verifies which tokens are still valid and delete old tokens. 
Note that this will soon be provided out-of-the-box as short-lived JWT tokens are currently being worked on

### Rate-limiting
For some users, being able to connect to the database from the frontend is not acceptable without IP-based rate-limiting. 
Session-based limiting can be implemented in FaunaDB as is shown in the [Rethinking twitter FQL example](https://css-tricks.com/rethinking-twitter-as-a-serverless-app/) we will provide another example that builds upon this one with such an implementation.
IP-based limiting is atm not possible to implement solely with FaunaDB, we will soon provide an example does rate-limiting using CloudFlare Workers. 

### HttpOnly cookies
If you require httpOnly cookies for XSS and/or protection against malicious users, the only thing you can do is to place serverless functions in between the calls. An 'edge' FaaS service like Cloudflare Workers is ideal in that case since they only bill CPU-time and are also distributed multi-region which would only add 10-20 ms of latency. 



