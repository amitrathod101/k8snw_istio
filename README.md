***Kubernetes Networking , CNI and Istio - Session 6 of APJ TAM App Modernization BootCamp***

Today we will learn all about Cluster Networking, 

Make sure Docker for Desktop is running. 

Open up a terminal and first make sure you dont have any old clusters running, if you do , then please delete them using 

    kind get clusters
    kind delete cluster --name tambootcamp

Create the KinD cluster using 

    kind create cluster --name tambootcamp

Give it some time, make sure the nodes status shows Ready using 

    kubectl get nodes

Before we go into Cluster Networking, have a look at the output of this command

    kubectl get all -n kube-system 

Note that the core-dns is a replicaset and that is why you see two pods.

>ClusterIP

First up will be the default ClusterIP. 

Let's create a sample deployment using the YAML file in the repo. 

    kubectl apply -f 01hello-deploy.yaml 

Now we will expose this using a Service using the YAML file. 

    kubectl apply -f 01hello-svc.yaml

The service name is “hello” and the type is “ClusterIP”.  The port specified is 80 and targetPort is http (which is 80).

Get a list of the pods running using 

    kubectl get pods 

Now copy any one of the backend pods name and describe it. 

    kubectl describe pod backend-6b7bc99459-58hdl

Note the port in there is the container port which is 80, we are exposing that using a service called as hello as a Cluster IP with a port 80 again. 

Let's port-forward it 

    kubectl port-forward pod/backend-6b7bc99459-58hdl 8080:80

Now use another terminal to curl it. 

    curl localhost:8080

You will get the message hello. 

So that's a Service of the type ClusterIP. Let's take a look at NodePort now. But before that let's delete the svc which we created. 

    kubectl delete svc hello

>NodePort

For NodePort type, we have another Service file in there called as 01hello-svc2.yaml. Let's apply that. 

    kubectl apply -f 01hello-svc2.yaml 

Have a look at that file, you will see that that major difference is that the type is a NodePort and we have specified port as 30001. You have to always specify a high(>30000) node port number. 

The idea of a NodePort is that K8S exposes the application on each of the K8S nodes that make up the cluster, so that it is accessible via NODE-IP:NODEPORT-IP. Whereas for ClusterIP, it will be accessible via the ClusterIP which you get when you do a kubectl get svc. 

>Istio

Now let's take a look at Istio

In your home directory, download Istio using 

    curl -L https://istio.io/downloadIstio | sh -

Since we dont have a Load Balancer for our lab, we will tweak our setup a bit. 

Navigate to istio-1.10.0/manifests/profiles and there you will see a file called as demo.yaml. 

We have to edit that demo.yaml file to include a NodePort for http and https traffic like 

    - port: 80
      name: http2
      nodePort: 31080 //add this line
      targetPort: 8080
    - port: 443
      name: https
      nodePort: 31443 //add this line
      targetPort: 8443



**Now let's install Istio**

Before doing the installation, I would recommend you to delete the current cluster using

    kubectl delete cluster --name tambootcamp

For the istio deployment, let's use a 3 node KinD Cluster. You can deploy the same using the kind-config.yaml file. 

    kind create cluster --name tambootcamp --config kind-config.yaml

This will deploy a 3 node cluster. Give it some time and check the status of Nodes using 

    kubectl get nodes

Once the status is Ready for all the three, we can start the installtion. 

If not in your istio installation location, navigate there. 

    cd istio-1.10.0

Now setup the path so istioctl would work.  

    export PATH=$PWD/bin:$PATH

Let's install it now: 

    istioctl install -f manifests/profiles/demo.yaml 

Add a default label to the default namespace for istio-injection.

    kubectl label namespace default istio-injection=enabled

Here's a sample application called BookInfo where we will test Istio Capabilities, see https://istio.io/latest/docs/examples/bookinfo/ for more information. 

    kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

Now we apply the YAML for creating the IngressGateway and the VirtualService, these show how the mappings for various parts of the overall website will work. 

    kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

Without any mutual TLS going on, we setup 4 destination rules , one for each of productpage, reviews, details and ratings. 

    kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml

To make sure things are fine on the istio side, you can run the command for validation checks. 

    istioctl analyze

>*Octant* - https://github.com/vmware-tanzu/octant

With that let's start octant and look at the services and pods coming up.
I would recommend using a new Terminal Tab or a Window for this. 

On Windows, you can use 

    choco install octant --confirm 

On mac , you can use

    brew install octant

That's it for octant's install. 

Start Octant by typing(on another terminal tab): 

    octant

With a great Clarity UI, it shows almost everything on the Cluster.  I love Octant(it's got a dark theme too)! 

Under Discovery and Load Balancing, you will see services, you will find the istio-ingressgateway service. Click through that and you will see a *"Start Port-Forward"* button. Click on it! It will give you a link, click on it again and you will be able to see the BookInfo App. 

We just did a Port-Forward here. Istio is not at play here so far. 

Keep an eye on the Reviews side and refresh the page, you will see the Star ratings change. Do it again, it changes again. This is because the ratings is coming from three different versions.

Now close that browser tab and head back to Octant and click on *Stop Port-Forward*

Remember how we modified the demo.yaml file earlier to include NodePorts, we will now see the application by using that NodePort at 

    http://localhost:31080/productpage

Refresh the page couple of times to see how the reviews are changing. You can notice that traffic is getting routed to various versions of the pods. In real production like scenarios, think of this as various versions of your code-revisions to which you can divert traffic. 

Now let's say you want only the v1 version of traffic to be hitting the application. You can modify the virtual Service using 

    kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

Now refresh the page and see that it doesnt change. 

Now let's say if we want the traffic to be a 50/50 split amongst the versions. For that we will delete the virtualservice which we just applied and apply a new one. 

    kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml

Now let's apply another virtualservice

    kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml

Have a test on the page, you will see a mix. 

There are many use cases of a ServiceMesh like Istio, namely A/B Testing, Blue-Green Testing or Canary deployments. They all mean the same where in you control how much traffic you want to pass on to one version of the software. 

Great! Now let's put some addons on it to again graphically visualize our bookinfo microservice application through *kiali*. 

When you downloaded istio , it also downloaded files for kiali, prometheus, jaeger, grafana, zipkin etc. 
All of these above need a separate 2 hour session on fully understanding it. 

Among these all, kiali is one of the coolest app's to work with. See it for your self. 

>**kiali**

Assuming you are in the istio-1.10.0 folder. You can run 

    kubectl apply -f samples/addons

You will see that some of these throw some errors, that's a dependency issue. You can easily work around that by running the same command again. 

    kubectl apply -f samples/addons

Ok, that should be better. Have a look at the status of the kiali deployment rollout status using 

    kubectl rollout status deployment/kiali -n istio-system

Once it says successfully rolled out, you can start kiali by using the command

    istioctl dashboard kiali

This should open up a webpage on your default browser, have a look around, look at the graphs, how everything is connected. 

Since you've tried kiali. There are other cool addons to be tried and are super popular in the opensource world. Try out prometheus, grafana, zipkin etc. 

Since you applied everything that was within the addons folder. You can use octant and go to those services and do a port-forward on them to see what they look like. 

Prometheus: https://prometheus.io/docs/prometheus/latest/getting_started/ 

Grafana: https://grafana.com/oss/grafana/ 

Jaeger: https://www.jaegertracing.io/docs/1.22/getting-started/ 

Zipkin: https://zipkin.io/pages/quickstart.html 

That's it for today. Have a good one! 

