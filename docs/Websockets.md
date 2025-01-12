## Real Time Communication Using WebSockets

The application leverages WebSockets to create a tunnel between the UI/Client and the backend, using which messages can be transmitted by both parties, to both parties. This eliminates the need to refresh a page every time an update is received to the backend.

### Architecture Details

The notification service leverages the following AWS services:

1. Amazon APIGateway’s WebSocket API
2. Lambda functions for connection management and message publishing
3. DynamoDB to store & maintain all connections. The table has the following index configuration:
   1. `connection_id` as the partition key
   2. `UserIdIndex` as the Global Secondary Index with `user_id` as the partition key. This index is used to fetch the list of connections for a particular `user_id`

### Understanding Notifications Service
The notifications service consists of 2 parts:

1. Connection management
2. Publishing messages to the interested connections

We can create routes for a WebSocket API (synonymous with API routes in REST API) and the RTC handler has the following routes:

1. $connect - This route is called when the client is requesting to connect with the backend. The `user_id` is passed as a query param to be stored in the connection management DynamoDB table
2. $disconnect - This route is called when the client disconnects from the backend. This is also called when the user navigates to a different page other than the scheduling page or when the tab is closed
3. $default - This is the default route, called when the request is for a route that is not configured

### Connection Management

The client calls the WebSocket API with the `$connect` route and with the user_id passed in the query param. API Gateway will call the RTC Handler lambda with the `$connect` route with the `connection_id`. 

The lambda function stores the following information in DynamoDB:

```json
{
  "connection_id": "M9aA1e3uvHcCH1A=",
  "user_id": "8a938091-7b1b-b753-017b-3f25f7540509"
}
```

The client calls the $disconnect flow when the user navigates to a different page or the tab is closed. The lambda function just deletes the connectionId from the DynamoDB data store

### [Sending messages to the client](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-data-from-backend.html)

Messages can be sent to the connected clients using the WebSocket’s connection URL and passing the `connection_id` to whom the message needs to be sent.

The publisher lambda does the following:
1. Fetches the `connection_id` for the corresponding `user_id`
2. send the message using the AWS SDK for API Gateway, which leverages the `connection_id` and automatically sends the message

### Factors to consider while using WebSockets
1. Cost - Websockets have a few factors considered for the pricing:
   1. We are charged for every minute a connection is kept alive
   2. The number of messages sent in the connection
2. The connection timeouts, quotas, and limits:
   1. 500 connections per second
   2. Message Payload size: 128 KB
   3. Idle Connection Timeout - 10 minutes
   4. Connection duration for WebSocket API	- 2 hours

### Steps to Create a WebSocket API

Navigate to the API Gateway service on the AWS console and click on `Create API`, select `Build` under the `WebSocket API` section and provide the following details in each section:

1. Step 1 (API Details)
   1. API Name - Name of the API we want to create. Ex: `awesome-food-app`
   2. Route selection expression - API Gateway uses the route selection expression to determine which route to invoke when a client sends a message. You can provide the standard `$request.body.action`
2. Step 2 (Add Routes) - configure the list of routes you want to add for the API. You can just add the `Predefined Routes`:
   1. `$connect` - triggered when a client connects to your API
   2. `$disconnect` - triggered when either the server or the client closes the connection
   3. `$default` - triggered if the route selection expression can't be evaluated against the message or if no matching route is found
   and click on `Next`
3. Step 3 (Attach Integrations) - this step is used to configure the target for the routes configured in the previous section. For every route configured, select the `Integration Type` as `Lambda` and provide the name of the notifications service lambda function name created above
4. Step 4 (Add Stages) - configure stage name for your WebSocket API. Ex: `dev`
5. Step 5 (Review & Create) - review all the configuration details and click on `Create and Deploy` if everything looks good 

Once the API is created, you can del

   
