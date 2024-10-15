# Pod
The "Pod" will be our image for a K8 pod which will contain the following:
- code-server container
- Express WS proxy & API
- Security-related env variables
## code-server container
See code-server.md. The image will be ran on pod startup. Yes, this is docker-within-docker. We need to do this because we need a way to commit and save the entire code-server image in order to save workspaces.
## Express WS proxy
We will be proxying the inner code-server container's exposed endpoint to our own pod's exposed endpoint.
### On WS open
Verify that the provided token matches the pod's token env variable.
### On WS close
Begin kill procedure
### On keepalive failure
Begin kill procedure
### On lifespan > 12 hr
Begin kill procedure
### On no connection > 1 minute
Begin kill procedure
## Express endpoints
### POST /kill
- Stop and commit the image.
- Send a request to the Agent PUT /image, with the provided tarball.
### GET /health
- Return a health response.