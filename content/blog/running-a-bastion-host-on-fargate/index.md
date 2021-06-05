---
title: Running a bastion host/jumpbox on Fargate
date: "2020-02-07T09:13:15.284Z"
description: Getting rid of EC2 instances even for SSH tunneling 
---

> Check out https://port7777.com, a product that was born out of
> this post! It's cheap and will setup everything you need to
> connect to your RDS from your local machine.

Where I work we strive to keep long-term maintenance burden as 
small as possible. We're a small team that sells SaaS to enterprise-grade
companies. Some of these very large brand names only sign a SaaS
contract with us after extensive data security questionnaire and
vetting process. We learned that it not only increase our chances,
but also make it easier to answer these questions with "It's AWS responsibility".

#### Bastion / Jumpbox

When using [AWS VPC](https://blog.deleu.dev/aws-vpc-for-software-engineers/)
it's common to have your RDS, among other resources, behind a private
subnet with a NAT Gateway to the internet. That means the outside world,
our office included, cannot initiate a request to these resources.
A bastion host tries to mitigate this issue by creating a point of
entry inside AWS VPC with a public IP. We can then connect to the
RDS by tunneling all of the network communication through the bastion.

A bastion host is a machine running an OpenSSH service that can
act as a tunnel between your requests and the desired resource.
For instance, a service such as Elasticache or RDS running inside
a VPC with a network address of `10.0.0.8` can only be accessed by
compute resources running inside the VPC itself. The bastion will
receive all of our requests that should be forwarded to Elasticache
or RDS and deliver us the response back. 

#### Why Fargate?

With the increasing popularity of serverless and the context in which I
work, having an EC2 running on our AWS account means we need to patch
the operating system for security as well as provide legal documents
as proof that we have this process in place. It costs more money to
handle these processes than the small fee that Fargate incur above
EC2 pricing. By using Fargate and getting rid of all of our EC2, our
security footprint is reduced to Docker images and the actual software
running in the machine. It also helps the sales pitch because these
large enterprises asking the questions know how AWS works and it helps
to establish trust.

#### What about drawbacks?

I've been working in the software industry long enough to know that no
solution comes without it's downsides. Currently there is no way to
assign an ENI to a Fargate instance, nor I think it would be possible.
Fargate is a service designed for scaling containers and we could not
have 2 containers taking the same IP address. Besides that, if the
container ever goes down and ECS starts a new one, it will have a new
IP address. To mitigate that problem, we would like to have a DNS
associated with the container so that we don't have to worry about
container fluctuation. But that requirement bring it's own limitations:
Network Load Balancer.

We already pay for an Application Load Balancer, but it cannot handle
TCP connections that are not HTTP/HTTPS. We would need to pay for 
an additional Network Load Balancer just so that we can have a DNS
associated with the NLB so that the NLB can establish a connection with
the Fargate container for us. This seemed rather wasteful as we'd be
using 0.01% of an NLB without any other use case for it.

After almost giving up on this, I had a nice little idea to try and
overcome this limitation. We can have a sidecar on ECS that updates
a Route 53 entry everytime a new container comes up. This would allow
us to have a reliable-ish DNS without the need for a NLB.

#### The implementation

Alpine Linux has become a big part of our organization. After running
security scanners on standard Docker images (PHP, Python, NodeJS, etc),
we noticed that most of them comes with vulnerability out of the box.
By designing our own Alpine images, we keep things as slim down as
possible that can pass security scanners in it's sleep.

```dockerfile
FROM alpine:3.11

RUN apk add openssh

RUN ssh-keygen -f /etc/ssh/ssh_host_rsa_key -N '' -t rsa

RUN echo "root:$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 36 ; echo '')" | chpasswd

COPY authorized_keys /root/.ssh/authorized_keys

COPY sshd_config /etc/ssh/sshd_config

RUN chmod 0700 ~/.ssh \
&&  chmod 0600 ~/.ssh/authorized_keys

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

This is a very small and simple Docker image that will run the
OpenSSH Daemon service on port 22. We generate a key and put a 
random password on `root` so that OpenSSH can properly work.
The important aspect is the `authorized_keys` that will have
public keys allowed to tunnel through this container. For the SSHD
configuration we can strengthen a few things.

```text
PasswordAuthentication no
PermitEmptyPasswords no

Match User root
  AllowTcpForwarding yes
  X11Forwarding no
  AllowAgentForwarding no
  ForceCommand /bin/false
```  

These settings will configure the service to only allow private key
authentication and will disable terminal access altogether. It can
only be used as a tunnelling mechanism for other resources inside
the VPC. 

Finally, the cherry of the cake is the sidecar.

```dockerfile
FROM alpine:3.11

RUN apk add --no-cache curl jq python groff less py-pip \
&&  pip --no-cache-dir install awscli \
&&  apk del py-pip

COPY dns.json /root/dns.json

COPY init.sh /root/init.sh

RUN chmod +x /root/init.sh

WORKDIR /root

CMD ["/root/init.sh"]
```

```shell script
#!/bin/sh

sed -i "s/!NEW_IP!/$(curl ipinfo.io | jq $data.ip)/g" /root/dns.json \
&& sed -i "s/!REGION!/${REGION}/g" /root/dns.json \
&& aws route53 change-resource-record-sets --hosted-zone-id ${HOSTED_ZONE} --change-batch file:///root/dns.json
```

```json
{
  "Comment": "Update Route 53 Bastion DNS",
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "my-service-name.!REGION!.mydomain.com",
      "Type": "A",
      "TTL": 600,
      "ResourceRecords": [{ "Value": !NEW_IP!}]
    }}]
}
```

These 3 files configure all that's necessary for the sidecar to work.
We have `!REGION` variable because we run the same setup in multiple
AWS regions and we use `curl` to grab our own ip address on start-up.
The AWS CLI allows for updating Route 53.

It's important to note that the lack of an NLB makes it so that we 
cannot scale this solution. However, given the fact that this is
about a jumpbox used by developers to access VPC-related AWS resources,
we do not need to scale this service. 1 container per region is more
than enough. One may also argue that when the container is being
replaced, DNS caching might become an issue, but the truth is we
expect to replace this container once every 6 months with the release
of a new Alpine Linux and if it crashes, we can get back to it soon
enough. 

#### Conclusion

I strive to keep things as simple as possible while providing security,
stability and costs optimized. It's very likely that my company would
be willing to pay for the NLB anyway to improve our security footprint,
but I wanted to provide a simple and robust solution for the problem
without any wasteful cost.

AWS ECS has integration with Route 53 for Service Discovery, but that 
is limited to private IP addresses only. Until they provide the same
for public IP, this solution will do just fine. It also doesn't hurt
to have an extremely slim down version of a jumpbox available for
tunneling only.

Hope you enjoyed the reading. If you have any questions,
send them my way on [Twitter](https://twitter.com/deleugyn).

Cheers.
