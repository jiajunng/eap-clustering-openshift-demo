EAP S2I clustering demo on OpenShift
============================================================
## Background
This demo shows how to deploy a clustered JBoss EAP running OpenShift with a simple web application that demonstrates session replication across the cluster.

## Requirements

- Running OpenShift cluster (Tested on v4.2)

## Basic instructions

To enable session clustering in your web application, EAP has to be configured (done by default in OpenShift) as well as adding the following to your src/main/webapp/WEB-INF/web.xml:

```<distributable/>```

- Create a project in OpenShift:

  ```$ oc new-project eap-clustering-demo```

- Provide EAP Access to Kubernetes API:

  JBoss EAP image is configured to use KUBE_PING which uses Kubernetes API to discover clustering members. Out of the box, KUBE_PING is the pre-configured and supported protocol.  

- Authorization must be granted to the service account the pod is running under to be allowed to access Kubernetes' REST api:

  ```$ oc policy add-role-to-user view system:serviceaccount:$(oc project -q):default -n $(oc project -q)```

- It is also posible to run EAP using a service account:

  ```$ oc policy add-role-to-user view system:serviceaccount:$(oc project -q):eap-service-account -n $(oc project -q)```

- Create the application:
     ```
     $ oc new-app --template=eap72-basic-s2i \
       -p APPLICATION_NAME=eap-app \
       -p SOURCE_REPOSITORY_URL=https://github.com/jiajunng/session-replication.git \
       -p SOURCE_REPOSITORY_REF='master' \
       -p CONTEXT_DIR='/' -l name=eap-app 

- Scale the deployment configuration so that two pods will running our EAP application: 
    ```
    $ oc scale dc/eap-app --replicas=2
    
- In the log of the EAP pod, you can see 2 pods formed a cluster:
    ```
    $ oc logs eap-app-1-7gbp8  | grep 'new cluster view'
    03:56:06,162 INFO  [org.infinispan.CLUSTER] (thread-12,ee,eap-app-1-7gbp8) ISPN000094: Received new cluster view for channel ee: [eap-app-1-7gbp8|1] (2) [eap-app-1-7gbp8, eap-app-1-dbw4x]
    03:56:06,168 INFO  [org.infinispan.CLUSTER] (thread-12,ee,eap-app-1-7gbp8) ISPN000094: Received new cluster view for channel ee: [eap-app-1-7gbp8|1] (2) [eap-app-1-7gbp8, eap-app-1-dbw4x]
    03:56:06,170 INFO  [org.infinispan.CLUSTER] (thread-12,ee,eap-app-1-7gbp8) ISPN000094: Received new cluster view for channel ee: [eap-app-1-7gbp8|1] (2) [eap-app-1-7gbp8, eap-app-1-dbw4x]
    
- Browse to the counter application through a route which have been created already:
    ```
    $ oc get routes
    NAME      HOST/PORT                                                              PATH   SERVICES   PORT    TERMINATION     WILDCARD
    eap-app   eap-app-eap-clustering-demo.apps.cluster-753a.sandbox311.opentlc.com          eap-app    <all>   edge/Redirect   None
  
- The web application displays the following information: 
  IMAGE
  - Session ID
  - Session ```counter``` and ```timestamp``` (Variables stored in the session)
  - The container name that the web page and session is being hosted from

- Click on the Increment Counter button

- To test that the session is being replicated, kill the running container:
   ```
   $ oc get pods
   NAME               READY   STATUS      RESTARTS   AGE
   eap-app-1-build    0/1     Completed   0          7m6s
   eap-app-1-cnzgg    1/1     Running     0          4m46s
   eap-app-1-deploy   0/1     Completed   0          4m55s
   eap-app-1-mnwts    1/1     Running     0          3m55s
   $ oc delete pod eap-app-1-cnzgg
   pod "eap-app-1-cnzgg" deleted

- Click the Refresh button, you should be able to see the same web session being served up from a new container.
