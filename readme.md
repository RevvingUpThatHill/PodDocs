# Pod alternatives
I propose to replace gitpod with a self-host dockerized code-server containers delivered via a reverse proxy.

https://github.com/coder/code-server

Code-server is a browser based VSCode that functions effectively identically to gitpod. The code-server delivers a single instance of the IDE (with a sudo access terminal) on a websocket, which is crucial info for the project specs below. The code-server project is open source and regularly maintained by the coder organization. This is the best solution because other solutions typically require a cost per-user with some obtuse auth method (such as github for gitpod) or is otherwise a subscription method that would be way too expensive (redhat devbox). Because we are already building this infra for labs, I would also like to use the pods for general training coding, so that everyone uses the same environments without the need for setup. This would reduce a the burden of getting environments running for the training which usually takes a couple of days.

## Here are the project components that I am envisioning:

### Pod
- A single instance of code-server which is running in a docker container.

### Agent
- An API (Spring) which receives requests to spin up or spin down pods and sends them to the nodes controllers. The Agent may also handle some load balancing and autoscaling concerns. My reasoning for handling this ourselves instead of using an out-of-the-box solution is due to statefulness, explained later.

### Node controller
- An API (Spring) replicated on each VM which receives requests from the Agent to manage the VM's pod containers.

### Proxy
- NGINX reverse proxy for connections to a pod after it has already been started.

## Monoworkspace or polyworkspace?

- Code-server has the advantage of managing separate workspaces as file paths that can be accessed through different URLs without any issues. That simplifies the process of saving a user's lab contents as well as maintaining installs and environment variables across sessions. The user may also have several labs and personal projects open simultaneously without compute cost. The same pod may be reused for the entirety of the user's lifecycle at Revature. We can also just make the user's personal coding workspaces a separate file path (in fact, creating a "personal use" workspace could just be a lab with blank contents).

## Labs
Labs can be retrieved using the cloud storage method we are currently using, with the added benefit of being able to restrict access to the lab contents to only the IPs of the Nodes. We can also replace the CLI command used by the user with a bash command invoked on the container by the node controller, so the lab may be instantly loaded once the pod container is ready. Because we are continuing to use VSCode, we should be able to continue using the current testing extension.

## Compute
- The base resource usage of an idling pod should be estimated at about .2 VCPU & .5 GB RAM. Of course, this will be expected to spike when any code is ran, and we could cap containers at 1 VCPU & 2GB RAM.

## Security

- We will require auth from revpro to spin up a pod.
- We will secure web traffic by using the WSS protocol.
- We will use the sysbox docker runtime to securely permit docker-in-docker within the pod.
- Code-server can also solve one of the biggest problems of maintaining our own pod infra - making access to the pods secure. Ideally, we could auth with a password as a query param, as was attempted in this rejected pull request - https://github.com/coder/code-server/pull/6734, and we'll need to make this contribution to the project ourselves. Revpro will receive this password from the agent when a pod spins up, and the agent will start the pod with the specified password.
- The destination node IP, as well as any query parameters to be sent to the pod, may also be obfuscated and decrypted at the NGINX layer using a JS or Lua module. If additional security is desired, we could also encrypt them on a rotating key which is reset on the NGINX module on an interval.

## Spinup

- The request to spin up a pod will come from revpro to the pod agent. whenever a user opens a lab. If the user has ever opened up their pod before, we'll pull their lab image from cloud storage and run it.

## Spindown

- We can detect socket events for when a user disconnects from the socket, allowing us to maintain the lifecycle of the pod automatically, and also autokill the pod on a timer. Every code-server will take up a significant amount of compute, so we should stop the containers when we detect a disconnection. I haven't tested if NGINX can handle this behavior as its ability to send requests is limited: if we can't do this, I would recommend (short term) simply killing all pods on a timer, or making a contribution to the code-server project that would allow us to ping it for active connections.
- After the container is stopped, we can use docker to save the stopped container as a tar, send the tar to cloud storage, and run the tar again on spinup.

## Orchestration

- The most significant problem for us to solve is the management of multiple containers. Our natural instinct is to use Kubernetes, but there is a key deficiency in that system: we have no access to stopped pods. We can not move the contents of a pod to cloud storage. That means that our only solution to running in Kubernetes is to keep all pods running all the time, which at the scale of PEP is probably not feasible. I tried bypassing these issues by running the pods as a sysbox DinD container, or by saving all the user's work in volumes, and I think that this would likely be significantly more work than just running the orchestration ourselves on VMs, since Kubernetes abstractions became more of a obstacle than a benefit, since our use case is extremely niche. We can manage this using any cloud SDK. Actually, GitPod themselves have decided the same, and they recently published an article on why the company felt that Kubernetes development for pods was no longer feasible: https://www.gitpod.io/blog/we-are-leaving-kubernetes
- Managing our own orchestration/scaling is a daunting task, but our real objective here is not to create a particularly efficient orchestration, but just to make saving pods feasible. So, for the POC, I would not even worry about ever scaling down nodes (save for perhaps a rare case where a node happens to have zero pods) and focus only on scaling up. Scaling down would require complicated eviction behavior which is not provided by an existing autoscaling solution since our application is extremely stateful (ie we would need to either wait for all pods to stop or gracefully stop them), but scaling up can be done without issue.
![image](./architecture.png)