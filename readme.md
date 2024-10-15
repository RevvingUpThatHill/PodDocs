# Pods
Documentation repo for the Pods project.
Our first iteration components will be:
- The "Agent", which manages the pod and image creation and destruction.
- The "PodProxy", which manages VSCode as a docker-in-docker container, and manages the proxy to the container, including actions to be taken on specific socket events like open and close.
- The "Code-Server", which is the actual VSCode environment users will access.
- "NGINX" will manage the proxying of sockets between users and pods.
# Labs
We also need to manage lab access at some point in the future. It is prudents that we first ensure that the basic pod infrastructure works first, after which we can start obtaining labs by unpacking labs as compressed blobs.