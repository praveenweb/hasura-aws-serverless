## Create a Hasura Cloud Project

- Click on the following button to create a new free project on Hasura Cloud:

<a href="https://cloud.hasura.io/?pg=aws-serverless-summit&plcmt=body&tech=default" target="_blank"><img src="https://graphql-engine-cdn.hasura.io/assets/main-site/deploy-hasura-cloud.png" /></a>

## Setup Amazon RDS PostgreSQL

- Login to the [AWS Console](https://console.aws.amazon.com/console/home).
- Create a new database with AWS RDS and select PostgreSQL.
- Allow public access and assign a VPC security group.
- Configure Hasura Cloud IP in inbound rules.
- Database URL format `postgresql://<user-name>:<password>@<public-ip>:<postgres-port>/<db>`
- Connect to Hasura Cloud

## Setup Amazon Cognito with JWT

- [Create user pools](https://docs.aws.amazon.com/cognito/latest/developerguide/getting-started-with-cognito-user-pools.html)
- Add an app client and note down the client ID.
- Configure app client settings, callback and signout URLs, enable Implicit Grant Flow for JWT
- Choose a domain name
- Hosted UI page - `https://your_domain/login?response_type=token&client_id=your_app_client_id&redirect_uri=http://localhost:3000/callback`

### Add Custom JWT Claims for Hasura

- Navigate to [AWS Lambda](https://console.aws.amazon.com/lambda/home)
- Create a function
- Copy the following handler code to generate custom claims

```javascript
exports.handler = (event, context, callback) => {
    event.response = {
        "claimsOverrideDetails": {
            "claimsToAddOrOverride": {
                "https://hasura.io/jwt/claims": JSON.stringify({
                    "x-hasura-user-id": event.request.userAttributes.sub,
                    "x-hasura-default-role": "user",
                    // do some custom logic to decide allowed roles
                    "x-hasura-allowed-roles": ["user"],
                })
            }
        }
    }
    callback(null, event)
}
```

- In Cognito, under Triggers, configure `Pre Token Generation` handler and select the lamdba function we just created above.
- Head to App Client Settings and click on `Launch Hosted UI`. Signup with a user and copy the id_token portion.
- Test the JWT in the debugger of [jwt.io](https://jwt.io)

### Configure Hasura Cloud ENV

- Copy the following config for `HASURA_GRAPHQL_JWT_SECRET` env.

```JSON
{
    "type":"RS256",
    "jwk_url": "https://cognito-idp.<aws-region>.amazonaws.com/<user-pool-id>/.well-known/jwks.json",
    "claims_format": "stringified_json"
}
```

Substitute the aws-region and user-pool-id from the URL parameters / General settings

### Create permissions for the role user

- Head to the table permissions tab, create a new role called `user` and apply a filter for `id` column to map to `x-hasura-user-id`.

## Set up Lambda for Hasura Events

- Create a simple function on Lambda.
- Add a route on API Gateway to expose the function outside.
- Add the endpoint to Hasura events to test an Event Trigger on a database table.
