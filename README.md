Pithos in Kubernetes
====================

This builds a Docker image for [Pithos](https://github.com/exoscale/pithos) and it also provides Kubernetes pods, replication controller and service to run Pithos in Kubernetes.

Run Cassandra with the Kubernetes [example](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/examples/cassandra)

## Running Cassandra in Kubernetes

You can use the Kubernetes example straight up or clone my own repo:

    $ git clone https://github.com/how2dock/dockbook.git
    $ cd ch05/examples

Then launch the Cassandra replication controller, increase the number of replicas and launch the service:

    $ kubectl create -f ./cassandra/cassandra-controller.yaml
    $ kubectl scale --replicas=4 rc cassandra
    $ kubectl create -f ./cassandra/cassandra-service.yaml

Once the image is downloaded you will have your Kubernetes pods in running state. Note that the image currently used comes from the Google registry. That's because this image contains a Discovery class specified in the Cassandra configuration. You could use the Cassandra image from Docker hub but would have to put that Java class in there to allow all cassandra nodes to discover each other. As I said, almost painless !

    $ kubectl get pods --selector="name=cassandra"

Once Cassandra discovers all nodes and rebalances the database storage you will get something like:

    $ ./kubectl exec cassandra-5f709 -c cassandra nodetool status
    Datacenter: datacenter1
    =======================
    Status=Up/Down
    |/ State=Normal/Leaving/Joining/Moving
    --  Address    Load       Tokens  Owns (effective)  Host ID                               Rack
    UN  10.16.2.4  84.32 KB   256     46.0%             8a0c8663-074f-4987-b5db-8b5ff10d9774  rack1
    UN  10.16.1.3  67.81 KB   256     53.7%             784c8f4d-7722-4d16-9fc4-3fee0569ec29  rack1
    UN  10.16.0.3  51.37 KB   256     49.7%             2f551b3e-9314-4f12-affc-673409e0d434  rack1
    UN  10.16.3.3  65.67 KB   256     50.6%             a746b8b3-984f-4b1e-91e0-cc0ea917773b  rack1
 
Note that you can also access the logs of a container in a pod with _kubectl logs_ very handy.

## Launching Pithos S3 object store

[Pithos](http://pithos.io) is a daemon which "provides an S3 compatible frontend to a cassandra cluster". So if we run Pithos in our Kubernetes cluster and point it to our running Cassandra cluster we can expose an S3 compatible interface.

To that end I created a Docker image for Pithos _runseb/pithos_ on Docker hub. Its an automated build so you can check out the Dockerfile there. The image contains the default configuration file. You will want to change it to edit your access keys and bucket stores definitions. I launch Pithos as a Kubernetes replication controller and expose a service with an external load balancer created on Google compute engine. The Cassandra service that we launched earlier allows Pithos to find Cassandra using DNS resolution. To bootstrap pithos we need to run a non-restarting Pod which installs the Pithos schema in Cassandra. Let's do it:

    $ kubectl create -f ./pithos/pithos-bootstrap.yaml

Wait for the bootstrap to happen, i.e for the Pod to get in _succeed_ state. Then launch the replication controller. For now we will launch only one replicas. Using an rc makes it easy to attach a service and expose it via a Public IP address.

    $ kubectl create -f ./pithos/pithos-rc.yaml
    $ kubectl create -f ./pithos/spithos.yaml
    $ ./kubectl get services --selector="name=pithos"
    NAME      LABELS        SELECTOR      IP(S)            PORT(S)
    pithos    name=pithos   name=pithos   10.19.251.29     8080/TCP
                                          104.197.27.250 

Since Pithos will serve on port 8080 by default, make sure that you open the firewall for public IP of the load-balancer.

## Use an S3 client

You are now ready to use your S3 object store, offered by Pithos, backed by Cassandra, running on Kubernetes using Docker. Wow...a mouth full !!!

Install [s3cmd](http://s3tools.org/s3cmd) and create a configuration file like so:

    $ cat ~/.s3cfg
    [default]
    access_key = AKIAIOSFODNN7EXAMPLE
    secret_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    check_ssl_certificate = False
    enable_multipart = True
    encoding = UTF-8
    encrypt = False
    host_base = s3.example.com
    host_bucket = %(bucket)s.s3.example.com
    proxy_host = 104.197.27.250 
    proxy_port = 8080
    server_side_encryption = True
    signature_v2 = True
    use_https = False
    verbosity = WARNING

Note that we use an unencrypted proxy (the load-balancer IP created by the Pithos Kubernetes service, don't forget to change it). The access and secret keys are the default stored in the [Dockerfile](https://github.com/runseb/pithos)

With this configuration in place, you are ready to use +s3cmd+:

    $ s3cmd mb s3://foobar
    Bucket 's3://foobar/' created
    $ s3cmd ls
    2015-06-09 11:19  s3://foobar

If you wanted to use Boto, this would work as well:

    #!/usr/bin/env python

    from boto.s3.key import Key
    from boto.s3.connection import S3Connection
    from boto.s3.connection import OrdinaryCallingFormat

    apikey='AKIAIOSFODNN7EXAMPLE'
    secretkey='wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'

    cf=OrdinaryCallingFormat()

    conn=S3Connection(aws_access_key_id=apikey,
                      aws_secret_access_key=secretkey,
                      is_secure=False,host='104.197.27.250',
                      port=8080,
                      calling_format=cf)

    conn.create_bucket('foobar')

And that's it. All of these steps make sound like a lot, but honestly it has never been that easy to run an S3 object store. Docker and Kubernetes truly make running distributed applications a breeze.


