# ECS Anywhere with AWS Copilot CLI Demo

## Goal

The goal of the article is to have a locally running VM, which is enrolled into an Amazon ECS cluster using [ECS
Anywhere](https://aws.amazon.com/ecs/anywhere/). On the local VM two containers will be running: a dummy web service
listening on a port and a load balancer, which discovers the web service and exposes it based on some predefined rules.

For deployment we will use the [AWS Copilot
CLI](https://aws.github.io/copilot-cli). AWS Copilot doesn't support ECS Anywhere as of writing this, however, its
flexible extensibility allows us to flip some CloudFormation properties and to have our containers running on 'external
container instances'.

The load balancer cannot be an Application Load Balancer in this case, as the goal is to have the load balancer _inside_
the VM. A great cloud-native alternative is [Traefik](https://doc.traefik.io/traefik/), which thanks to its combination
of static and dynamic configuration is a perfect fit for the task.

## Benefits of this architecture

Thanks to the building blocks the following features are available for the local cluster:

1. It runs on-prem (which can be a requirement due to external constraints)
2. The container orchestration is fully cloud-native (you can use all features of ECS and SSM)
3. Rolling updates are available for the backend services due to Traffik's dynamic discovery capabilities
4. Circuit-breaker is enabled and performs automated rollbacks, if the services fail to stabilize during deployment
5. You can make use of the expressive Copilot manifests to configure your application

## Prerequisites

- [Install Vagrant](https://developer.hashicorp.com/vagrant/docs/installation)
- [Install VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- [Install AWS Copilot CLI](https://aws.github.io/copilot-cli/docs/getting-started/install/)
- [Install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and
  [configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) the AWS CLI

## Step-by-step guide

1. Initialize Copilot app

   ```bash
   copilot init
   ```

2. Initialize & deploy Copilot `test` environment

   ```bash
   copilot env init -n test
   copilot env deploy -n test
   ```

3. Start Vagrant machine

   ```bash
   vagrant up
   ```

4. Get registration command for the local VM as an external machine:
   1. Navigate to the ECS cluster in the [ECS Service](https://console.aws.amazon.com/ecs/v2/clusters)
   2. Choose the right region
   3. Click the _Infrastructure_ tab, then the _Register external instances_ button
   4. In the popup set the _Number of instances_ to 1, and for _Instance role_ select _Create new_.
   5. Copy the registration command for Linux in the last step

5. Register the local VM as an external machine:

   ```bash
   vagrant ssh
   sudo -s
   # And then execute command copied from step 4
   # curl ... 
   ```

6. Make sure that the registration completes and that the instance shows up in the _Infrastructure_ tab of the cluster.

7. Initialize & deploy the workloads

   ```bash
   copilot svc init -e test -n whoami
   copilot svc deploy -e test -n whoami
   copilot svc init -e test -n traefik
   copilot svc deploy -e test -n traefik
   ```

8. Now you can navigate to the Traefik dashboard to check if the Docker provider is enabled and the services are discovered:

   ```plain
   http://localhost:8080/
   ```

9. You can get a response from the whoami service by making the following request:

   ```bash
   curl -H 'Host: whoami.domain.com' http://localhost:8081/whoami
   ```

   Note: the port forwarding for the host port 80 with VirtualBox didn't work for me, so I redirected the VM's port 80
   to port 8081 on the host.

## Clean-up

1. De-register the external instance
   1. Navigate to the ECS cluster in the [ECS Service](https://console.aws.amazon.com/ecs/v2/clusters)
   2. Choose the right region
   3. Click the _Infrastructure_ tab
   4. Under _Container Instances_ select your VM, then _Actions_ and _Drain_
   5. Wait until the instance disappears (this can take a while)
2. Delete the Copilot application:

   ```bash
   copilot app delete
   ```

3. Delete the managed node from SSM
   1. Open [SSM Fleet Manager](https://console.aws.amazon.com/systems-manager/fleet-manager/managed-nodes)
   2. Select your VM
   3. Press _Node actions_, then _Node settings_, then _Deregister this managed node_

4. Delete the IAM role `ecsExternalInstanceRole`

5. Destroy the local machine

   ```bash
   vagrant destroy
   ```

## References

- The article [Implementing application load balancing of Amazon ECS Anywhere workloads using Traefik
  Proxy](https://aws.amazon.com/blogs/containers/implementing-application-load-balancing-of-amazon-ecs-anywhere-workloads-using-traefik-proxy)
  demonstrates a similar use case, however it suggests using the ECS discovery with Traefik, which enables putting the
  Traefik load balancer on one external machine and placing the backend service on a different host. It's not exactly
  what I was looking for, additionally, it has a major drawback: it requires the exposure of the port of the backend
  containers on the host, which can be a security risk, additionally it prevents rolling updates and horizontal scaling,
  as the port can only be bound to one container at a time.
- [AWS Copilot CLI docs](https://aws.github.io/copilot-cli/)
- [Traefik docs](https://doc.traefik.io/traefik/)
  - [Traefik Docker dynamic configuration
    reference](https://doc.traefik.io/traefik/reference/dynamic-configuration/docker/)
  - [Traefik static CLI configuration](https://doc.traefik.io/traefik/reference/static-configuration/cli/)
