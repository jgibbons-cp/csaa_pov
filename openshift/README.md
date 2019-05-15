# Instructions for deploying Datadog in OpenShift

The Datadog OpenShift integration is located [here](https://docs.datadoghq.com/integrations/openshift/).  Datadog’s Kubernetes integration supports OpenShift versions 3.3 and up. No additional setup is required for OpenShift-specific tags to be collected.  NOTE:  if you are 
using Atomic OS you may get limited metrics since it is a read-only OS.

Cluster resource quota metrics are collected by the leader Agent. Configure the Agent event collection and leader election in order to send metrics to Datadog.

The Datadog Kubernetes documentation is located [here](https://docs.datadoghq.com/agent/kubernetes/).
 
The instructions used to create the datadog-agent.yaml are located [here](https://docs.datadoghq.com/agent/kubernetes/daemonset_setup/?tab=dockersocket).

If your Kubernetes has role-based access control (RBAC) enabled, configure RBAC permissions for your Datadog Agent service account.  Instructions can be found for this [here](https://docs.datadoghq.com/agent/kubernetes/daemonset_setup/?tab=dockersocket#configure-rbac-permissions).

I started with the [base yaml](https://docs.datadoghq.com/agent/kubernetes/daemonset_setup/?tab=dockersocket#create-manifest).  I will outline what changes I made and what is still required.

NOTE: Replace <YOUR_API_KEY> with your Datadog API key or use Kubernetes secrets to set your API key as an environment variable. If you opt to use Kubernetes secrets, refer to Datadog’s instructions for setting an API key with Kubernetes secrets. Consult the Docker integration to discover all of the configuration options.
Security Note: please remove any API keys before checking-in yaml.

1) \# Uncomment this section to use Kubernetes secrets to configure your Datadog API key  The API key for the account can
 be found [here](https://app.datadoghq.com/account/settings#api) under API Keys.
 
2) I uncommented \# hostPort: 8126 so the agent will listen for APM traces

3) \# Kubernetes Secrets - uncomment this section to supply API Key with secrets  The API key for the account can
 be found [here](https://app.datadoghq.com/account/settings#api) under API Keys.
 
4) I set the DD_LOGS_ENABLED and DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL variable to true in your env section.  This can be
 found [here](https://docs.datadoghq.com/agent/kubernetes/daemonset_setup/?tab=dockersocket#log-collection).
 
 The volume type is [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath).
 
5) Mount the Docker socket or /var/log/pods to collect logs.  I commented out the docker socket lines.

6) I mounted the pointdir volume in volumeMounts which is used to store a file with a pointer to all the containers that the Agent is collecting logs from. This is to make sure none are lost when the Agent is restarted, or in the case of a network issue.

7) I added the APM and Distributed tracing yaml as documented [here](https://docs.datadoghq.com/agent/kubernetes/daemonset_setup/?tab=dockersocket#apm-and-distributed-tracing).

8) I enabled the ability to send StatsD traffic to Datadog as noted [here](https://docs.datadoghq.com/agent/kubernetes/daemonset_setup/?tab=dockersocket#dogstatsd).

9) I enabled live process collection as noted [here](https://docs.datadoghq.com/graphing/infrastructure/process/?tab=kubernetes#installation).

In order to begin tracing, point your application-level tracers to where the Datadog Agent host is using the environment variable DD_AGENT_HOST      
<br>The documentation for Java can be found [here](https://docs.datadoghq.com/tracing/setup/java/#installation-and-getting-started).  

In addition to adding the following JVM argument when starting your application in your IDE, Maven or Gradle application script, or java -jar command

`-javaagent:/path/to/the/dd-java-agent.jar`

the following system properties or environment variables need to be set (NOTE: this assumes we will be tracing PAS as we did
with the non-OpenShift environment):

-Ddd.trace.analytics.enabled=true   
-Ddd.jmxfetch.enabled=true  
-Ddd.trace.global.tags=env:perf1   
-Ddd.trace.header.tags=X-dynatrace:loadrunner,user-agent:user-agent   
-Ddd.jms.analytics.enabled=true  
-Ddd.logs.injection=true  
-Ddd.trace.span.tags="JVM Name:<jvm_name>"

Bala is using the dd.trace.methods as well so you may ask him for that one.

or the environment variables noted [here](https://docs.datadoghq.com/tracing/setup/java/#configuration).

When ready you can deploy the DaemonSet with:

`kubectl create -f datadog-agent.yaml`

To verify the Datadog Agent is running in your environment as a DaemonSet, execute:

`kubectl get daemonset`

If the Agent is deployed, you will see output similar to the text below, where DESIRED and CURRENT are equal to the number of nodes running in your cluster.

`NAME            DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE  
 datadog-agent   2         2         2         2            2           <none>          16h`
