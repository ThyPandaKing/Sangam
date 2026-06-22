I am planning to build an integration platform, which can be used to integrate multiple platforms and perform actions on top of them.

I will start mentioning requirements one by one:

1. Integration
- Should have multiple integrations with different technologies
- example: ServiceNow, Slack, WhatsApp, Teams
- Adding new integration should not be difficult (should be an option from client side)
- Every user should be able to create a new integration for their own usecase with their own requirements


2. Requests:
- Once we have integrations available, a user should be able to activate their own integration
- Once activate it should have 2 type of request methods:
- Incoming: Request coming into the platform (our platform should expose endpoints)
- outgoing: Platform making the request
- Requests should be configurable from client side

3. Flow
- Every active integrations by a user can be wrapped inside a "Flow" that can be triggred based on a schedule or based on any incoming request
- Flow should be able to handle multiple "requests" (active integrations for each user)

4. Action
- We should support actions at each step like Email, data transform
- Action should have input and output
- Should be extendable, we can start with the above mentioned 3 features
- Flow will basically be a queue of "actions"

5. IAM and RBAC
- As we will have multiple users, we need IAM and RBAC for our app

6. AI agent
- We will use AI chat option as well to:
- Build new integrations via chat
- Build new flows and actions
- Request can be parsed to do this (use tools to make them)
- Analysing the existing flows and its outputs

7. Logs: for each flow
8. Home: For each user it should show the running flows and draft flows


Non functional requirements:

1. Scale to multiple users and integrations running at same time
2. We can use SQL database for Integrations, IAM, Credentials and NoSQL for Request and Actions (suggest any other better alternatives)
3. Incoming request should be added to a queue and should be broadcasted to all the "active" flows taking that input


Your task:

1. Start writing a project plan with estimated timeline to finish this by End of year 2026
2. Project plan should be technical and have proper details of what to use, how to use, why to use, cost of using that.
3. I am planning to learn a lot of new technolgies from this, let me list down my high level design, check that out and then suggest/analyse it. inform me of each desicion you make and why you want to make it

HLD (by me):

1. API gateway for incoming request: Ratelimiting, logs, IAM
2. Microservies architecture:
Let's keep 2 for now
- first for creating static data (which is not running like making flows, integrations, login, granting access etc)
- Second for running flows, this is where the execution of inbound and outbound request will happen

3. DB
- we need 2 db, SQL for fixed schema kind of data, and NoSql for changing data

4. Server
- Need to host multiple servers based on scale, should have load balancers and auto-scaling

5. Queue/Streaming platform:
- We should queue incoming request there and it should be streamed to each flow

6. Cache: we use redis to store frequent data

7. Analytics: Need to store analytics for users/organisations


Just create the planning document for now, we don't need to work on code