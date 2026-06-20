## INSTALL DOCKER

A very detailed instructions to install Docker are provide in the below link

https://docs.docker.com/get-docker/ 

You can create an Ubuntu EC2 Instance on AWS and run the below commands to install docker.

```
sudo apt update
sudo apt install docker.io -y
```


### Start Docker and Grant Access

A very common mistake that many beginners do is, After they install docker using the sudo access, they miss the step to Start the Docker daemon and grant acess to the user they want to use to interact with docker and run docker commands.

Always ensure the docker daemon is up and running.

A easy way to verify your Docker installation is by running the below command

```
docker run hello-world
```

If the output says:

```
docker: Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/create": dial unix /var/run/docker.sock: connect: permission denied.
See 'docker run --help'.
```

This can mean two things, 
1. Docker deamon is not running.
2. Your user does not have access to run docker commands.


### Start Docker daemon

You use the below command to verify if the docker daemon is actually started and Active

```bash
sudo systemctl status docker
```

If you notice that the docker daemon is not running, you can start the daemon using the below command

```bash
sudo systemctl start docker
```


### Grant Access to your user to run docker commands

To grant access to your user to run the docker command, you should add the user to the Docker Linux group. Docker group is create by default when docker is installed.

``` bash
sudo usermod -aG docker ubuntu
```

In the above command `ubuntu` is the name of the user, you can change the username appropriately.

**NOTE:** : You need to logout and login back for the changes to be reflected.

## The next steps are neccesary to avoid debricated warning on WSL You can skip it you are on VM
```
Prepare your system to accept secure packages over HTTPS
```
```bash
sudo apt install -y ca-certificates curl gnupg
```
Download Docker’s official cryptographic key so your system can safely verify the authenticity of the packages
```bash
sudo install -m 0755 -d /etc/apt/keyrings
```
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
Inject the official Docker repository into your system's package source list
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Refresh your package catalog and successfully install the plugin
```bash
sudo apt update
```
```bash
sudo apt install -y docker-buildx-plugin
```
Verify the plugin's presence by running:
```bash
docker buildx version
```
#### This will print your active version profile, and your subsequent docker build executions will automatically use the modern engine without throwing legacy builder deprecation alerts.
### Docker is Installed, up and running

Use the same command again, to verify that docker is up and running.

```bash
docker run hello-world
```

Output should look like:

---
Hello from Docker!
This message shows that your installation appears to be working correctly.

### Great Job, Now start with the examples folder to write your first Dockerfile and move to the next examples. Happy Learning

