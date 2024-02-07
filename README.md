# Bastion pod for accessing VPC resources

The idea is to get access from developer machine to a resource accesible to a kubernetes cluster VPC.
It uses a pod as a bastion and SSH-tunnel session to route traffic

The pod uses the image `linuxserver/openssh-server` and runs SSH at port 2222 to avoid requiring root privileges

The selected image generates at runtime the file `/etc/ssh/sshd_config` so injecting it in Kubernetes doesn’t work, because it is overridden when the image starts.
- So, in this example we modify the initial run scripts of the docker image to add the features we want during this config generation phase
- Port value defaults to 2222
- AllowTcpForwarding enabled using an env variable (this is not there in the original run script)
 

Connecting through SSH requires a pair of public/private keys
- Generate them locally before applying the deployment
    - First create a folder for the new keys `mkdir ./etc/ssh`
    - Using `ssh-keygen -A -f .` to generate host keys (empty passphrase) under the current folder

- Modify the configmap substituting `INSERT_HERE_YOUR_PUBLIC_KEY` with your key
- `ssh-tunnel sed "s/INSERT_HERE_YOUR_PUBLIC_KEY/$(cat ./etc/ssh/ssh_host_rsa_key.pub | sed -e 's/[\/&]/\\&/g')/g" ssh-tunnel-configmap.yaml > configmap.yaml`

# DNS resolution of destination servers
Some servers (like MongoDB Atlas via VPC peering ) validate the name of the `Host` header for the connection. Or we use a tool with hardcoded URLs. So in order to use the expected DNS name
- Since we will be connecting through localhost...
- Add an alias to `/etc/hosts`
- `127.0.0.1  my.server.com` 

# Connecting
Once the pod  is deployed, we can use it to tunnel through it with SSH

Expose the port SSH server to localhost
- `kubectl port-forward <ssh-pod-id> 2222:2222`
- `kubectl port-forward $(kubectl get pods |grep ssh-tunnel | head -1 |cut -d ' ' -f1) 2222:2222`
- This makes ssh available in localhost:2222

- If the remote service requires DNS validation, edit your /etc/hosts accordingly
    - e.g. to access a remote mongoDB server add this:
    - `127.0.0.1 my-cluster-pri.xyz.mongodb.net`
- Create the SSH tunnel
  - Using the private part of the public/private pair
  - ` ssh -i /tmp/key myuser@localhost -p 2222 -L 27016:my-cluster-pri.xyz.mongodb.net:27016`

Now, the target server can be reached, e.g to reach one Mongo Atlas server

```
mongosh "mongodb://$USER:$PASSWORD@my-cluster-pri.xyz.mongodb.net:27016/?ssl=true&authSource=admin&retryWrites=true&readPreference=secondary"
```
