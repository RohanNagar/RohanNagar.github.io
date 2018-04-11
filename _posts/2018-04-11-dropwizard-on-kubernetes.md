---
layout: post
title:  "Running Dropwizard Applications on Kubernetes"
author: "Rohan Nagar"
---

[Dropwizard](http://www.dropwizard/io) is a popular Java library for building RESTful web services.
I am using this library for my user management application, [Thunder](https://github.com/RohanNagar/thunder),
and it makes building the API much easier and more enjoyable. When it comes to deploying applications,
[Kubernetes](https://kubernetes.io/) is the new hot product. I wanted to run my Dropwizard application on
Kubernetes, and decided to put together a guide for anyone else looking to do the same.

# Building the Application

I will assume that if you're reading this guide, you already have a Dropwizard project ready to deploy.
If not, follow the [getting started](http://www.dropwizard.io/1.3.1/docs/getting-started.html) guide for
v1.3.1 of Dropwizard. You can also check out my project, [Thunder](https://github.com/RohanNagar/thunder),
for a working example that you can use. Just clone that repository to your computer using git.

Using Maven, build the application into a packaged jar that you can run anywhere.

```
$ mvn package
```

# Create the Docker image

In order to run your application on Kubernetes, you need a Docker image with your application.
So, let's build one! Create a new file named `Dockerfile` in the base directory of your project
(i.e. next to your parent `pom.xml`) and add the following content.

{% highlight docker  linenos %}
FROM openjdk:8-jre-alpine

LABEL maintainer "YOUR NAME youremail@example.com"

COPY ./application/target/application-*.jar application.jar

EXPOSE 80 81
ENTRYPOINT ["java", "-jar", "/application.jar", "server", "/home/config/config.yaml"]
{% endhighlight %}

Let's go through this file line by line. First, we specify what existing Docker image to build this
image `FROM`. We are using the `openjdk:8-jre-alpine` image since we are running our application using
Java 8. If you are using a different Java version,
check the [available images](https://hub.docker.com/_/openjdk/). We use the Alpine flavor in order to
have a smaller Docker image.

Next, we define the maintainer `LABEL` which provides your information. This line is optional.

Then, in order to get your application into the Docker image, we need to `COPY` it in. This line copies
the jar that we built into the image. If your jar file is named differently, you will need to modify this
line.

After copying the application, we have to `EXPOSE` the ports that our application will be listening on. If you
set up your Dropwizard configuration to listen on different ports, change those here.

Finally, we define the `ENTRYPOINT` for running the image. This is the command to start your
Dropwizard application. Notice that we reference `config.yaml`, which does not yet exist in our image.
We will take care of adding the configuration file when we deploy the application. We don't want to bake it
into the image because our configuration could change at any time.

Now, let's build our image. Make sure you have Docker installed.
If you don't, go [here](https://docs.docker.com/install/) to install.
Run the following command to tag and build your image.

```
$ sudo docker build -t yourname/application:test .
```

Feel free to change the tag to anything else. You should replace `yourname` with your Docker Hub username,
and replace `application` with the name of your application.

Now we will push our image to Docker Hub. If you haven't yet, create an account on [Docker Hub](https://hub.docker.com/). You can also use a private Docker registry, but I will not cover that in this post. Once you create an account, login on the command line.

```
$ sudo docker login
```

After entering your credentials, you're set to push your image!

```
$ sudo docker push yourname/application:test
```

Check your Docker Hub account to verify that your application was pushed! Now that we have our application
packed into a Docker image, we can run it using Kubernetes.

# Create a Kubernetes Cluster

Kubernetes is growing quickly in popularity, and there are many great guides on how to [get started](https://kubernetes.io/docs/tutorials/kubernetes-basics/).
If you are not familiar with Kubernetes (or K8s), I recommend that you first learn about it and make sure
that you are comfortable with the terminology.

To create a cluster, we can use [minikube](https://kubernetes.io/docs/getting-started-guides/minikube/) to
run one locally. First, we need to [install minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/).
Minikube requires a hypervisor (i.e. VirtualBox, Hyper-V, or VMWare Fusion) to be installed, so make sure that
you follow those directions. Install the hypervisor, then install the command line tools `kubectl`, then install
minikube. The instructions differ based on what OS you are on, so please follow the linked instructions.

Once everything is installed, start the cluster.

```
$ minikube start
```

If you do not wish to use minikube locally, and want to jump straight into running in the cloud, follow guides
for creating a K8s cluster on one of the following cloud providers:

- [Azure AKS](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough)
- [AWS using kops](https://github.com/kubernetes/kops/blob/master/docs/aws.md)
- [Google GKE](https://cloud.google.com/kubernetes-engine/docs/quickstart)

# Create Configuration File

Kubernetes supports [ConfigMaps](https://kubernetes-v1-4.github.io/docs/user-guide/configmap/) that
provide a way to inject your container with configuration data. This is the approach we will use to
provide the Dropwizard configuration file to our application.

First, create a new file called `application-config.yaml`, and add the following content.

{% highlight yaml  linenos %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  config.yaml: |
    server:
      applicationConnectors:
        - type: http
          port: 80
      adminConnectors:
        - type: http
          port: 81
{% endhighlight %}

This is a YAML file that defines a Kubernetes resource. The `apiVersion` tells Kubernetes what version
of the API to use when processing the file. The `kind` of resource that we will create with this file
is a ConfigMap. In the `metadata`, we define the name of the ConfigMap, and in the `data`, we define
what the configuration actually is. In this case, we want the configuration to be a file called
`config.yaml`. Inside that file, we want to define our Dropwizard configuration. Put your Dropwizard
YAML configuration here. For this example, I have just defined the `server` configuration for Dropwizard,
but you can see [here](https://github.com/RohanNagar/thunder/blob/master/scripts/kubernetes/thunder-config.yaml)
for a full example based on my Thunder application.

Now that we have the file, we can create the ConfigMap in our running Kubernetes cluster. Make sure that
you are connected to your cluster using `kubectl`, and then run the command.

```
$ kubectl apply -f application-config.yaml
```

You can then verify that it was created by running the get command.

```
$ kubectl get configmap
```

You should see the name of your ConfigMap as the output.

# Running the Application

Now, we're ready to deploy our application. Define another file called `application-deployment.yaml`,
and add the following content.

{% highlight yaml  linenos %}
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: application
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: application
    spec:
      containers:
        - name: application
          image: yourname/application:test
          imagePullPolicy: Always
          ports:
            - containerPort: 80
            - containerPort: 81
          volumeMounts:
            - name: config-volume
              mountPath: /home/config
          livenessProbe:
            httpGet:
              path: /healthcheck
              port: 81
            initialDelaySeconds: 10
            timeoutSeconds: 1
          readinessProbe:
            httpGet:
              path: /healthcheck
              port: 81
            initialDelaySeconds: 10
            timeoutSeconds: 1
      volumes:
        - name: config-volume
          configMap:
            name: application-config
---
apiVersion: v1
kind: Service
metadata:
  name: application
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: application
{% endhighlight %}

This is where everything comes together. We first define a Kubernetes Deployment, which is the main application.
Inside the `spec`, we define that we want 2 `replicas` of the application running at all times. Then, inside
the `template` spec, we define the containers we want to run and the volumes that we want to create. For the
containers, we only want to run one container - the Docker image that we built earlier. We define the image
tag from earlier under the `image` property. We also have to tell Kubernetes that we have a few open `ports`
on the container, `80` and `81`. We also define a `volumeMount` that mounts a volume into the container.
This volume will contain our Dropwizard configuration file, and so we specify the path as `/home/config`, since
that is the path we assumed the configuration would exist in our `ENTRYPOINT` command in our `Dockerfile`.
Finally, we define a `livenessProbe` that Kubernetes will probe to ensure our application is alive (and replace
it if it is dead), and a `readinessProbe` that Kubernetes will probe before starting to serve traffic to that
Pod. In both cases, we tell Kubernetes to probe the Dropwizard `/healthcheck` endpoint on the admin port,
port `81`.

Lastly, we define a volume called `config-volume` that holds the ConfigMap we created earlier. As mentioned in the paragraph above, we mount this volume into our container so that our application has access to the data.

The second resource that is defined is a Service. This acts as a load balancer, and Kubernetes will manage the
relationship between our multiple running containers and the load balancer to spread traffic between them.
We can specify the open ports on the load balancer, and what the `targetPort` should be (the port to proxy
the traffic to on the container). The `selector` defines which set of containers to balance across, so we
use the `app: application` label that we defined in the Deployment.

Note that the Service type `LoadBalancer` only works in the context of clusters running on cloud providers. If
we are using minikube, we should use the `NodePort` type to expose the service on the IP of the running Node.

Let's create the Deployment and Service in one go:

```
$ kubectl apply -f application-deployment.yaml
```

And your application is up! If you are using the LoadBalancer, wait a few minutes and then run the following
command to get the external IP of the load balancer.

```
$ kubectl get svc application
```

Once you have the external IP, run a test using `curl`. For example, if your application has a `/hello`
endpoint:

```
$ curl 56.77.45.34/hello
```

If everything worked, you will get your HTTP response back! If it doesn't work, you can troubleshoot by
checking to make sure your containers are running:

```
$ kubectl get deployment application
```

If they are not running, you should check the Dropwizard logs to see what went wrong. Get the name of one of
your Pods:

```
$ kubectl get pods 
```

Find one of the `application-*` Pods (for example, application-6cffc7bb7-gjg5q), and then run:

```
$ kubectl logs application-6cffc7bb7-gjg5q
```

This will show you all of the logs output by Dropwizard.

# Conclusion

Now you've learned how to containerize your Dropwizard application, keep your configuration separate,
and create a Kubernetes ConfigMap and Deployment for your application. Deploying your Dropwizard application
in this way will allow you to iterate and deploy quickly and effortlessly. Enjoy your containerized Dropwizard
application, and reach out (rohannagar11@gmail.com) or leave a comment if you have any problems or questions.

