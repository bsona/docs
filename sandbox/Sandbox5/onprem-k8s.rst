.. _s5-k8s:

***************************************
Learn Netris operations with Kubernetes
***************************************

.. contents:: 
   :local: 

Intro
=====
This Sandbox environment provides an existing Kubernetes cluster that has been deployed via `Kubespray <https://github.com/kubernetes-sigs/kubespray>`_. For this scenario, we will be using the `external LB <https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ha-mode.md>`_ option in Kubespray. A dedicated Netris L4LB service has been created in the Sandbox Controller to access the k8s apiservers from users and non-master nodes sides.

.. image:: /images/sandbox-l4lb-kubeapi.png
    :align: center

To access the built-in Kubernetes cluster, put "Kubeconfig" file which you received by the introductory email into your ``~/.kube/config`` or set "KUBECONFIG" environment variable ``export KUBECONFIG=~/Downloads/config`` on your local machine. After that try to connect to the k8s cluster:

.. code-block:: shell-session

  kubectl cluster-info

The output below means you've successfully connected to the sandbox cluster:

.. code-block:: shell-session

    Kubernetes master is running at https://api.k8s-sandbox5.netris.io:6443

    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

Install Netris Operator
=======================

The first step to integrate the Netris Controller with the Kubernetes API is to install the Netris Operator. Installation can be accomplished by installing regular manifests or a `helm chart <https://github.com/netrisai/netris-operator/tree/master/deploy/charts/netris-operator>`_.  For this example we will use the Kubernetes regular manifests:

1. Install the latest Netris Operator:

.. code-block:: shell-session

  kubectl apply -f https://github.com/netrisai/netris-operator/releases/latest/download/netris-operator.yaml

2. Create credentials secret for Netris Operator:

.. code-block:: shell-session

  kubectl -nnetris-operator create secret generic netris-creds \
  --from-literal=host='https://sandbox5.netris.io' \
  --from-literal=login='demo' --from-literal=password='Your Demo user pass'

3. Inspect the pod logs and make sure the operator is connected to Netris Controller:

.. code-block:: shell-session

  kubectl -nnetris-operator logs -l netris-operator=controller-manager --all-containers -f

Example output demonstrating the successful operation of Netris Operator:

.. code-block:: shell-session

  {"level":"info","ts":1629994653.6441543,"logger":"controller","msg":"Starting workers","reconcilerGroup":"k8s.netris.ai","reconcilerKind":"L4LB","controller":"l4lb","worker count":1}

.. note::
  
  After installing the Netris Operator, your Kubernetes cluster and physical network control planes are connected. 

Deploy an Application with an On-Demand Netris Load Balancer
============================================================

In this scenario we will be installing a simple application that requires a network load balancer: 

Install the application `"Podinfo" <https://github.com/stefanprodan/podinfo>`_:

.. code-block:: shell-session

  kubectl apply -k github.com/stefanprodan/podinfo/kustomize

Get the list of pods and services in the default namespace:

.. code-block:: shell-session

  kubectl get po,svc

As you can see, the service type is "ClusterIP":

.. code-block:: shell-session

  NAME                           READY   STATUS    RESTARTS   AGE
  pod/podinfo-576d5bf6bd-7z9jl   1/1     Running   0          49s
  pod/podinfo-576d5bf6bd-nhlmh   1/1     Running   0          33s
  
  NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
  service/podinfo      ClusterIP   172.21.65.106   <none>        9898/TCP,9999/TCP   50s

In order to request access from outside, change the type to "LoadBalancer":

.. code-block:: shell-session

  kubectl patch svc podinfo -p '{"spec":{"type":"LoadBalancer"}}'

Check the services again:

.. code-block:: shell-session

  kubectl get svc

Now we can see that the service type has changed to LoadBalancer, and "EXTERNAL-IP" switched to pending state:

.. code-block:: shell-session

   NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
   podinfo      LoadBalancer   172.21.65.106   <pending>     9898:32584/TCP,9999:30365/TCP   8m57s

Going into the Netris Controller web interface, navigate to **Services → L4 Load Balancer**, and you may see L4LBs provisioning in real-time. If you do not see the provisioning process it is likely because it already completed. Look for the service with the name **"podinfo-xxxxxxxx"**

.. image:: /images/sandbox-podinfo-prov.png
    :align: center

After provisioning has finished, let's one more time look at service in k8s:

.. code-block:: shell-session

  kubectl get svc

You can see that "EXTERNAL-IP" has been injected into Kubernetes:

.. code-block:: shell-session
  
  NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                         AGE
  podinfo      LoadBalancer   172.21.65.106   50.117.59.141   9898:32584/TCP,9999:30365/TCP   9m17s

Let's try to curl it (remember to replace the IP below with the IP that has been assigned in the previous command):

.. code-block:: shell-session

  curl 50.117.59.141:9898

The application is now accessible directly on the internet:

.. code-block:: json
  
  {
   "hostname": "podinfo-576d5bf6bd-nhlmh",
   "version": "6.0.0",
   "revision": "",
   "color": "#34577c",
   "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
   "message": "greetings from podinfo v6.0.0",
   "goos": "linux",
   "goarch": "amd64",
   "runtime": "go1.16.5",
   "num_goroutine": "8",
   "num_cpu": "4"
  }

As seen, "PodInfo" developers decided to expose 9898 port for HTTP, let's switch it to 80:

.. code-block:: shell-session

  kubectl patch svc podinfo --type='json' -p='[{"op": "replace", "path": "/spec/ports/0/port", "value":80}]'

Wait a few seconds, you can see the provisioning process on the controller:

.. image:: /images/sandbox-podinfo-ready.png
    :align: center

Curl again, without specifying a port:

.. code-block:: shell-session

  curl 50.117.59.141

The output is similar to this:

.. code-block:: json
  
  {
   "hostname": "podinfo-576d5bf6bd-nhlmh",
   "version": "6.0.0",
   "revision": "",
   "color": "#34577c",
   "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
   "message": "greetings from podinfo v6.0.0",
   "goos": "linux",
   "goarch": "amd64",
   "runtime": "go1.16.5",
   "num_goroutine": "8",
   "num_cpu": "4"
  }

You can also verify the application is reachable by putting this IP address directly into your browser.

.. topic:: Milestone 1

  Congratulations!  You successfully deployed a network load balancer and exposed an application from your cloud to the internet.  Time to get yourself an iced coffee.


Using Netris Custom Resources
=============================

Introduction to Netris Custom Resources
---------------------------------------

In addition to provisioning on-demand network load balancers, Netris Operator can also provide automatic creation of network services based on Kubernetes CRD objects. Let's take a look at a few common examples:

L4LB Custom Resource
--------------------

In the previous section, when we changed the service type from "ClusterIP" to "LoadBalancer", Netris Operator detected a new request for a network load balancer, then it created L4LB custom resources. Let's see them:

.. code-block:: shell-session

  kubectl get l4lb

As you can see, there are two L4LB resources, one for each podinfo's service port:

.. code-block:: shell-session

  NAME                                                            STATE    FRONTEND        PORT       SITE     TENANT   STATUS   AGE
  podinfo-default-66d44feb-0278-412a-a32d-73afe011f2c6-tcp-80     active   50.117.59.141   80/TCP     US/NYC   Admin    OK       33m
  podinfo-default-66d44feb-0278-412a-a32d-73afe011f2c6-tcp-9999   active   50.117.59.141   9999/TCP   US/NYC   Admin    OK       32m

You can't edit/delete them, because Netris Operator will recreate them based on what was originally deployed in the service specifications.

Instead, let's create a new load balancer using the CRD method.  This method allows us to create L4 load balancers for services outside of what is being created natively with the Kubernetes service schema.  Our new L4LB's backends will be "srv04-nyc" & "srv05-nyc" on TCP port 80. These servers are already running the Nginx web server, with the hostname present in the index.html file.

Create a yaml file:

.. code-block:: shell-session

  cat << EOF > srv04-5-nyc-http.yaml
  apiVersion: k8s.netris.ai/v1alpha1
  kind: L4LB
  metadata:
   name: srv04-5-nyc-http
  spec:
   ownerTenant: Admin
   site: US/NYC
   state: active
   protocol: tcp
   frontend:
     port: 80
   backend:
     - 192.168.45.64:80
     - 192.168.46.65:80
   check:
     type: tcp
     timeout: 3000
  EOF

And apply it:

.. code-block:: shell-session

  kubectl apply -f srv04-5-nyc-http.yaml

Inspect the new L4LB resources via kubectl:

.. code-block:: shell-session

  kubectl get l4lb

As you can see, provisioning started:

.. code-block:: shell-session

  NAME                                                            STATE    FRONTEND        PORT       SITE     TENANT   STATUS         AGE
  podinfo-default-d07acd0f-51ea-429a-89dd-8e4c1d6d0a86-tcp-80     active   50.117.59.141   80/TCP     US/NYC   Admin    OK             2m17s
  podinfo-default-d07acd0f-51ea-429a-89dd-8e4c1d6d0a86-tcp-9999   active   50.117.59.141   9999/TCP   US/NYC   Admin    OK             3m47s
  srv04-5-nyc-http                                                active   50.117.59.142   80/TCP     US/NYC   Admin    Provisioning   6s

When provisioning is finished, you should be able to connect to L4LB. Try to curl, using the L4LB frontend address displayed in the above command output:

.. code-block:: shell-session

  curl 50.117.59.142

You will see the servers' hostname in curl output:

.. code-block:: shell-session

  SRV04-NYC

You can also inspect the L4LB in the Netris Controller web interface:

.. image:: /images/sandbox-l4lbs.png
    :align: center

V-Net Custom Resource
---------------------

If one of the backend health-checks is marked as unhealthy like in the screenshot above, it means you didn't create "vnet-customer" V-Net as described in the :ref:`"Learn by Creating Services"<s5-v-net>` manual. If that's the case, let's create it from Kubernetes using the V-Net custom resource.

Let's create our V-Net manifest:

.. code-block:: shell-session

  cat << EOF > vnet-customer.yaml
  apiVersion: k8s.netris.ai/v1alpha1
  kind: VNet
  metadata:
   name: vnet-customer
  spec:
   ownerTenant: Demo
   guestTenants: []
   sites:
     - name: US/NYC
       gateways:
         - 192.168.46.1/24
       switchPorts:
         - name: swp2@sw22-nyc
  EOF

And apply it:

.. code-block:: shell-session

  kubectl apply -f vnet-customer.yaml

Let's check our V-Net resources in Kubernetes:

.. code-block:: shell-session

  kubectl get vnet

As you can see, provisioning for our new V-Net has started:

.. code-block:: shell-session

  NAME            STATE    GATEWAYS          SITES    OWNER   STATUS   AGE
  vnet-customer   active   192.168.46.1/24   US/NYC   Demo    Active   10s

After provisioning has completed, the L4LB's checks should work for both backend servers, and incoming requests should be balanced between them. 

Let's curl several times to see that:

.. code-block:: shell-session

  curl 50.117.59.142

As we can see, the curl request shows the behavior of "round robin" between the backends:

.. code-block:: shell-session

  SRV05-NYC
  curl 50.117.59.142
  
  SRV05-NYC
  curl 50.117.59.142
  
  SRV05-NYC
  curl 50.117.59.142
  
  SRV04-NYC

.. note::

  *If intermittently the result of the curl command is "Connection timed out", it is likely that the request went to the srv05-nyc backend, and the "Default ACL Policy" is set to "Deny". To remedy this, configure an ACL entry that will allow the srv05-nyc server to communicate with external addresses. For step-by-step instruction review the* :ref:`ACL documentation<s5-acl>`.

BTW, if you already created "vnet-customer" V-Net as described in the :ref:`"Learn by Creating Services"<s5-v-net>`, you may import that to k8s, by adding ``resource.k8s.netris.ai/import: "true"`` annotation in V-Net manifest, the manifest should look like this:

.. code-block:: shell-session

  cat << EOF > vnet-customer.yaml
  apiVersion: k8s.netris.ai/v1alpha1
  kind: VNet
  metadata:
   name: vnet-customer
   annotations:
     resource.k8s.netris.ai/import: "true"
  spec:
   ownerTenant: Demo
   guestTenants: []
   sites:
     - name: US/NYC
       gateways:
         - 192.168.46.1/24
       switchPorts:
         - name: swp2@sw22-nyc
  EOF
  
Apply it:

.. code-block:: shell-session

  kubectl apply -f vnet-customer.yaml

After applying the manifest containing "import" annotation, the V-Net, created from the Netris Controller web interface, will appear in k8s and you will be able to manage it from Kubernetes.

.. code-block:: shell-session

  kubectl get vnet

  NAME            STATE    GATEWAYS          SITES    OWNER   STATUS   AGE
  vnet-customer   active   192.168.46.1/24   US/NYC   Demo    Active   7s
 
BGP Custom Resource
-------------------

Let's create a new BGP peer, that is listed in the :ref:`"Learn by Creating Services"<s5-e-bgp>`.

Create a yaml file:

.. code-block:: shell-session

  cat << EOF > iris-isp2-ipv4-customer.yaml
  apiVersion: k8s.netris.ai/v1alpha1
  kind: BGP
  metadata:
    name: iris-isp2-ipv4-customer
  spec:
    site: US/NYC
    hardware: SoftGate2
    neighborAs: 65007
    transport:
      name: swp16@sw02-nyc
      vlanId: 1052
    localIP: 50.117.59.86/30
    remoteIP: 50.117.59.85/30
    description: Example BGP to ISP2
    prefixListInbound:
      - permit 0.0.0.0/0
    prefixListOutbound:
      - permit 50.117.59.128/28 le 32
  EOF

And apply it:

.. code-block:: shell-session

  kubectl apply -f iris-isp2-ipv4-customer.yaml

Check created BGP:

.. code-block:: shell-session

  kubectl get bgp

Allow up to 1 minute for both sides of the BGP sessions to come up:

.. code-block:: shell-session

  NAME                      STATE     BGP STATE   PORT STATE   NEIGHBOR AS   LOCAL ADDRESS      REMOTE ADDRESS     AGE
  iris-isp2-ipv4-customer   enabled               Link Up      65007         50.117.59.86/30   50.117.59.85/30   15s

Then check the state again:

.. code-block:: shell-session

  kubectl get bgp

The output is similar to this:

.. code-block:: shell-session

  NAME                      STATE     BGP STATE                                       PORT STATE   NEIGHBOR AS   LOCAL ADDRESS      REMOTE ADDRESS     AGE
  iris-isp2-ipv4-customer   enabled   bgp: Established; prefix: 160; time: 00:01:27   Link Up      65007         50.117.59.86/30   50.117.59.85/30   2m3s

Feel free to use the import annotation for this BGP if you created it from the Netris Controller web interface previously.

Return to the Netris Controller and navigate to **Net → Topology** to see the new BGP neighbor you created.

Importing Existing Resources from Netris Controller to Kubernetes
-----------------------------------------------------------------

You can import any custom resources already created from the Netris Controller to k8s by adding the following annotation:

.. code-block:: yaml

  resource.k8s.netris.ai/import: "true"

Otherwise, if try to apply them without the "import" annotation, the Netris Operator will complain that the resource with such name or specs already exists.
 
After importing resources to k8s, they will belong to the Netris Operator, and you won't be able to edit/delete them directly from the Netris Controller web interface, because the Netris Operator will put everything back, as declared in the custom resources.

Reclaim Policy
--------------

There is also one useful annotation. So suppose you want to remove some custom resource from k8s, and want to prevent its deletion from the Netris Controller, for that you can use "reclaimPolicy" annotation:

.. code-block:: yaml

  resource.k8s.netris.ai/reclaimPolicy: "retain"

Just add this annotation in any custom resource while creating it. Or if the custom resource has already been created, change the ``"delete"`` value to ``"retain"`` for key ``resource.k8s.netris.ai/reclaimPolicy`` in the resource annotation. After that, you'll be able to delete any Netris Custom Resource from Kubernetes, and it won't be deleted from the Netris Controller.

.. seealso::

  See all options and examples for Netris Custom Resources `here <https://github.com/netrisai/netris-operator/tree/master/samples>`_.


Netris Calico CNI Integration
=============================

Netris Operator can integrate with Calico CNI, in your Sandbox k8s cluster, Calico has already been configured as the CNI, so you can try this integration. It will automatically create BGP peering between cluster nodes and the leaf/TOR switch for each node, then to clean up it will disable Calico Node-to-Node mesh. To understand why you need to configure peering between Kubernetes nodes and the leaf/TOR switch, and why you should disable Node-to-Node mesh, review the `calico docs <https://docs.projectcalico.org/networking/bgp>`_.

Integration is very simple, you just need to add the annotation in calico's ``bgpconfigurations`` custom resource. Before doing that, let's see the current state of ``bgpconfigurations``:

.. code-block:: shell-session

  kubectl get bgpconfigurations default -o yaml

As we can see, ``nodeToNodeMeshEnabled`` is enabled, and ``asNumber`` is 64512 (it's Calico default AS number):

.. code-block:: yaml

  apiVersion: crd.projectcalico.org/v1
  kind: BGPConfiguration
  metadata:
   annotations:
     ...
   name: default
   ...
  spec:
   asNumber: 64512
   logSeverityScreen: Info
   nodeToNodeMeshEnabled: true

Let's enable the "netris-calico" integration:

.. code-block:: shell-session

  kubectl annotate bgpconfigurations default manage.k8s.netris.ai/calico='true'

Let's check our BGP resources in k8s:

.. code-block:: shell-session

  kubectl get bgp

Here are our freshly created BGPs, one for each k8s node:

.. code-block:: shell-session

  NAME                                 STATE     BGP STATE                                      PORT STATE   NEIGHBOR AS   LOCAL ADDRESS      REMOTE ADDRESS      AGE
  iris-isp2-ipv4-customer              enabled   bgp: Established; prefix: 160; time: 00:06:18  Link Up      65007         50.117.59.86/30   50.117.59.85/30    7m59s
  sandbox5-srv06-nyc-192.168.110.66   enabled                                                               4230000000    192.168.110.1/24   192.168.110.66/24   26s
  sandbox5-srv07-nyc-192.168.110.67   enabled                                                               4230000001    192.168.110.1/24   192.168.110.67/24   26s
  sandbox5-srv08-nyc-192.168.110.68   enabled                                                               4230000002    192.168.110.1/24   192.168.110.68/24   26s  

You might notice that peering neighbor AS is different from Calico's default 64512.  The is because the Netris Operator is setting a particular AS number for each node.

Allow up to 1 minute for the BGP sessions to come up, then check BGP resources again:

.. code-block:: shell-session

  kubectl get bgp

As we can see, our BGP peers have become established:

.. code-block:: shell-session

  NAME                                 STATE     BGP STATE                                       PORT STATE   NEIGHBOR AS   LOCAL ADDRESS      REMOTE ADDRESS      AGE
  iris-isp2-ipv4-customer              enabled   bgp: Established; prefix: 160; time: 00:07:48   Link Up      65007         50.117.59.86/30   50.117.59.85/30    8m41s
  sandbox5-srv06-nyc-192.168.110.66   enabled   bgp: Established; prefix: 5; time: 00:00:44     N/A          4230000000    192.168.110.1/24   192.168.110.66/24   68s
  sandbox5-srv07-nyc-192.168.110.67   enabled   bgp: Established; prefix: 5; time: 00:00:19     N/A          4230000001    192.168.110.1/24   192.168.110.67/24   68s
  sandbox5-srv08-nyc-192.168.110.68   enabled   bgp: Established; prefix: 5; time: 00:00:44     N/A          4230000002    192.168.110.1/24   192.168.110.68/24   68s

Now let's check if ``nodeToNodeMeshEnabled`` is still enabled:

.. code-block:: shell-session

  kubectl get bgpconfigurations default -o yaml

It is disabled, which means the "netris-calico" integration process is finished:

.. code-block:: yaml

  apiVersion: crd.projectcalico.org/v1
  kind: BGPConfiguration
  metadata:
    annotations:
      manage.k8s.netris.ai/calico: "true"
      ...
    name: default
    ...
  spec:
    asNumber: 64512
    nodeToNodeMeshEnabled: false

.. note::

  Netris Operator won't disable Node-to-Node mesh until all BGP peers of all the nodes in the k8s cluster become established.

Finally, let's check if our earlier deployed "Podinfo" application is still working when Calico Node-to-Node mesh is disabled:

.. code-block:: shell-session

  curl 50.117.59.141

Yes, it works:

.. code-block:: json

  {
   "hostname": "podinfo-576d5bf6bd-mfpdt",
   "version": "6.0.0",
   "revision": "",
   "color": "#34577c",
   "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
   "message": "greetings from podinfo v6.0.0",
   "goos": "linux",
   "goarch": "amd64",
   "runtime": "go1.16.5",
   "num_goroutine": "8",
   "num_cpu": "4"
  }

Disabling Netris-Calico Integration
-----------------------------------

To disable "Netris-Calico" integration, delete the annotation from Calico's ``bgpconfigurations`` resource:

.. code-block:: shell-session

  kubectl annotate bgpconfigurations default manage.k8s.netris.ai/calico-

or change its value to ``"false"``.

.. topic:: Milestone 2

  Congratulations!  You completed Milestone 2.  Time to get yourself another iced coffee or even a beer depending on what time it is!
