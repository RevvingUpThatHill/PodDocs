# Agent
The "Agent" will be the service built to accept requests to provision new Pods.
# Endpoints
## POST /pod
Require: 
- User auth
Request body:
- {"lab": "name of the lab requested"}
Effect:
- Spin down all other pods currently used by the User
    - Send a POST /kill to all active pods belonging to User
    - Delete the existing nodeIP service
- Spin up new pod which will service the user's workspace.
    - Check if an existing image exists for the given lab requested.
    - Generate a token for the pod.
    - Run a new Pod instance.
    - Create a nodeIP service for the container.
    - Wait for the Pod's health endpoint to respond.
Responses:
- 201 OK
-   Body: { "id":1234, "ip":"1.2.3.4", "token":"1234ABC }
- 403 Forbidden
## GET /pod
Require:
- Admin auth
QueryParams:
- ?user: id of the user
Responses:
- 200 OK
-   Body: [{"id":1234, "ip":"1.2.3.4"} ...]
- 403 Forbidden
## DELETE /pod/{id}
Require:
- Admin auth
Effect:
- Send a POST /kill to the pod with ID
Status:
- 200 OK
-   Body: { "id":1234, "ip":"1.2.3.4" }
## POST /user
Require:
- Admin auth
Effect:
- Save a new user
Status:
- 200 OK
-   Body: { "id":1234, "username":"user1", "active":true }
## GET /user
docs todo
## GET /user/{id}
docs todo
## POST /login
docs todo
## PUT /image
Require:
- Valid pod token
Request body:
- {"token":"123ABC", "lab":"labname", "userId":1234, "image":"binary"}
Effect: 
- Persist the image provided in the binary to S3
- Perform PUT for image data on the provided labname and userId
## DELETE /image
Require:
- User auth
Request body:
- {"lab":"labname", "userId":1234}
Effect:
- Delete any saved workspace image