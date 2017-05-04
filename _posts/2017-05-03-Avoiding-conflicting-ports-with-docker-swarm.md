---
layout: post
title:  "Avoiding conflicting ports with Docker Swarm"
date:   2017-05-04
categories: docker swarm jenkins 
---

### TL;DR 

```
# Launch the service with a random port.
docker service create --name myService --publish 0:80 --replicas 3
# Retreive the port
{% raw %}docker service inspect --format="{{with index .Endpoint.Ports 0\}\}{{.PublishedPort}}{{end}}" myService{% endraw %}
```
  
### Long Version
  
At my current [employer](http://www.spotx.tv), we use Jenkins to run automated tests - 
these jobs deploy containerized applications to a cluster of nodes with 
Docker daemons running in swarm mode.
 
The tests themselves are run from the Jenkins slave on which the job is executing,
so the tests need to be able to access the service running on the
swarm cluster.

When you create a Swarm service you can specify a port to publish.  

```
docker service create --name myService --publish 1234:80 --replicas 3
```

In this case all requests to the swarm cluster on port 1234 will be load-balanced
and forwarded to port 80 on the service. We can tell our tests to hit
this port on any swarm node and we're done.

However, this job is triggered on every commit so it is run frequently. When more
than one commit happens within a short space of time you'll see errors like

```
Error response from daemon: rpc error: code = 3 desc = port '1234' is already in use by service
```

Jenkins is trying to deploy more than one service published on the same port.

To prevent this, we could use something like the [Throttle Builds Plugins](https://wiki.jenkins-ci.org/display/JENKINS/Throttle+Concurrent+Builds+Plugin)
to ensure only one of this type of service is deployed at once. That kinda sucks, in a busy project you might end up with long queues.

To get around this the Docker CLI allows you to specify a port of 0 to have the kernel allocate a free port.

```
docker service create --name myService --publish 0:80 --replicas 3
```

Retrieving which port has been allocated is a little more difficult, the [docker port command](https://docs.docker.com/engine/reference/commandline/port/)
lets you view port mappings for containers running on that machine but running in swarm mode
means you may not be on the same node the container is running on.

Luckily 'service inspect' and the power of [GoLang's Templates](https://golang.org/pkg/text/template/) allow retrieval
of the published port from any swarm node.

```
{% raw %}docker service inspect --format="{{with index .Endpoint.Ports 0\}\}{{.PublishedPort}}{{end}}" myService{% endraw %}
```

Inside a Jenkinsfile this might look something like

```
def get_port_from_swarm(String service_name){

    def swarm_manager = 'ssh -o StrictHostKeyChecking=no -l $USER $SWARM_HOST '
    def inspect = 'docker service inspect --format="{{with index .Endpoint.Ports 0}}{{.PublishedPort}}{{end}}" ' + service_name
    def command = swarm_manager + "'" + inspect +"'"
    
    sshagent(['$SWARMHOSTKEY']) {
        port = sh(
                returnStdout: true,
                script: command
        )
    }
    return port
}
```