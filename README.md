# Openshift Kubernetes Tests

Following an Openshift Kubernetes workshop for building and deploying a microservice architecture. The workshop shows you how OpenShift can be used for deploying stateless web applications and applications which require persistent file system storage. The flexibility makes OpenShift a platform for deploying both web applications and databases.

- Available in Python, Java, and NodeJS. This repository follows a Java based application.
- Uses a free [Openshift playgrounds](https://www.katacoda.com/courses/openshift/playgrounds/) using the Openshift web console with administrative access and `oc` terminal commands.
- Uses code from the [Redhat Roadshow parks repositories](https://github.com/openshift-roadshow/) on GitHub:
  - Front end service users a Docker image of the [parksmap web application](https://github.com/openshift-roadshow/parksmap-web/).
  - The backend services ares called [nationalparks](https://github.com/openshift-roadshow/nationalparks) and [mlbparks](https://github.com/openshift-roadshow/mlbparks).
  - It is a Java Spring Boot application that performs 2D geo-spatial queries against a MongoDB database to locate and return map coordinates of all National Parks and Major League Baseball parks in the world. The backends are webservices that return a JSON list of places.

## Workshop Walkthrough

This README follows the workshop using `oc` commands and the Developer perspective in the Openshift web console unless it is said otherwise and a GitHub repository for code. Instead of GitHub, a self hosted [Gogs](https://gogs.io/) server can be used.

### Deploy Container

In Openshift web console, choose topology / +Add > container image and enter

`quay.io/openshiftroadshow/parksmap:latest`

Choose application values:

```txt
Application Name : workshop

Name : parksmap
```

Uncheck the checkbox next to Create a route to the application, it will be set up manually later.

Choose labels:

First the name to be given to the application.

`app=workshop`

Next the name of this deployment.

`component=parksmap`

And finally, the role this component plays in the overall application.

`role=frontend`

Using oc command in terminal, check application is running and its configuration.

- The oc command can be downloaded from the web console under help (question mark icon) and then choose command line tools. The oc command uses `curl` to use the Kubernetes API behind the scenes.
- Keep the web console open on the Topology window of the application to see changes shown as changes are made

```sh
oc get pods
oc get pod <insert name of the container> -o yaml
oc get service parksmap -o yaml
oc describe service parksmap
oc get deployment
# Check replica set
oc get rs
```

### Scaling and Self Healing, ReplicaSet Management

Scale parksmap "application" up to 2 instances. We can do this with the `scale` command. You could also do this by incrementing the Desired Count in the OpenShift web console.

```sh
oc scale --replicas=2 deployment/parksmap
# Verify scaling
oc get rs
# Verify 2 pods are deployed
oc get pods
# Verify that the Service reflects two endpoints:
oc describe svc parksmap
# Alternative verification, note endpoints may have various IPs, Openshift's service manages the pods and IPs automatically
oc get endpoints parksmap
```

Experiment with deleting a pod and seeing a new one being created. Then scale back to 1 replicas.

```sh
oc delete pod parksmap-... && oc get pods
# Scale to 1, required steps for rest of workshop
oc scale --replicas=1 deployment/parksmap
```

### Exposing Application to the Internet

Expose a route to the application

```sh
# Check no routes exist first
oc get routes
# Get the Service name to expose
oc get services
# Create the route
oc create route edge parksmap --service=parksmap
# Verify route was created, see 8080-tcp was exposed which is from the image Dockerfile
# https://github.com/openshift-roadshow/parksmap-web/blob/master/Dockerfile
oc get route
```

This application is now available at the URL shown in the Developer Perspective > Topology. Click the link and you will see it.

### Exploring Logs

Check logs for a pod in the web console with Topology view > pod > Resources tab > View logs. Alernatively, using oc commands:

```sh
oc get pods
oc logs <insert pod name>
```

Notice there is an error in the logs which will be addressed later.

### Role-Based Access Control

The Openshift default service account is the one taking the responsibility of running the pods, and OpenShift uses and injects this service account into every pod that is launched. The permissions for that service account can be changed,

View current permissions in the web console, go to the Topology view in the Developer Perspective > pod detail > Details tab > Namespace > Role Bindings. The Create binding can be use to modify roles there.

Switch to the active project and give the default service account view access:

```sh
# Get list of projects
oc projects
oc project <insert project name>
oc policy add-role-to-user view -z default
```

The `oc policy` command above is giving a defined *role* (`view`) to a user. But we are using a special flag, `-z`. The `-z` syntax is a special one that saves us from having to type out the entire string, which, in this case, is `system:serviceaccount:<project name>:default`. The `-z` flag will only work for service accounts in the current project. If you're referring to a service account in a different project, use the `-n <project>`switch.

Redeploy the application as it has given up trying to query the API.

```sh
oc rollout restart deployment/parksmap
```

After restart, logs will no longer show errors related to the service account.

Optionally, grant access to another user to view your project

```sh
oc policy add-role-to-user view <insert other username>
```

### Remote Shell Session to a Container Using the CLI

Access the shell of the running container, for example to troubleshoot issues, debugging, or checking the container.

```sh
oc get pods
oc rsh <insert pod name>
# Alternate to above using bash shell
oc rsh <insert pod name> /bin/bash
# Check root files
sh-4.2$ ls /
```

After last command, you should see a shell prompt `sh-4.2$`. NOTE: The default shell used by `oc rsh` is `/bin/sh`. If the deployed container does not have **sh** installed and uses another shell, (e.g. **A Shell**) the shell command can be specified after the pod name in the issued command.

To execute commands, you can log into the shell as above or use the `oc exec` command. This example shows a JAR file:

`oc exec <insert pod name> -- ls -l /parksmap.jar`

The `--` syntax in the `oc exec` command delineates where exec's options end and where the actual command to execute begins. Take a look at `oc exec --help` for more details.

Alternatively, `oc rsh` can also be used in a similar way like:

`oc rsh <insert pod name> whoami`

Note the user is not the user specific in the `Dockerfile`. For security reasons, OpenShift does not run containers as the user specified in the Dockerfile by default. In fact, when OpenShift launches a container its user is actually randomized.

If you want or need to allow OpenShift users to deploy container images that do expect to run as root (or any specific user), a small configuration change is needed. You can learn more about the [container image guidelines](https://docs.openshift.com/container-platform/latest/openshift_images/create-images.html#images-create-guide-general_create-images) for OpenShift.

### Deploy the Java Code

#### Source to Image (S2I)

In Openshift web console, choose topology / +Add > From Git and enter.

- From Git Repo URL: `https://github.com/justintungonline/nationalparks.git`. The workshop uses an embedded git server. This walk through uses a GitHub repository instead
- Git Type: Other
- Choose builder image as Java, version 11
- Application Name : workshop
- Name : nationalparks

Expand the Labels section and add 3 labels:

The name of the Application group:

`app=workshop`

Next the name of this deployment.

`component=nationalparks`

And finally, the role this component plays in the overall application.

`role=backend`

Click create and check the pods and build status.

#### Java Application

This is a Java-based application that uses Maven as the build and dependency system. For this reason, the initial build will take a few minutes as Maven downloads all of the dependencies needed for the application.

```sh
oc get pods
oc get builds
# View build logs
oc logs -f builds/nationalparks-1
# Verify the build completed and the pod is in running state
oc get pods
# Notice the build pod is completed while the back end pod is now running

# Openshift automatically created a route for the application
# Check the routes and the URLs for each pod
oc get routes
```

The back end does not have a web interface; however, it will still respond to a browser. Visit the URL listed and add `/ws/info/` to it. You will see a JSON string like:
`{"id":"nationalparks","displayName":"National Parks","center":{"latitude":"47.039304","longitude":"14.505178"},"zoom":4}`  

The JSON confirms the back end is running

### Adding a Database (MongoDB)

#### Templates and Create a Database

Create a MongoDB template inside the project, so that is only visible to our user and we can access it from Developer Perspective to create a MongoDB instance.

`oc create -f https://raw.githubusercontent.com/openshift-labs/starter-guides/ocp-4.6/mongodb-template.yaml -n <project name>`

In the Developer Perspective click **+Add** and then **Database**. In the Databases view, you can click **Mongo** to filter for just MongoDB (Make sure to uncheck **Operator Backed** option from **Type** section). Click **Instantiate Template** button
Alternatively, you could type `mongodb` in the search box. Once you have drilled down to see MongoDB, find the **MongoDB (Ephemeral)** template and select it. Normally, in a real application persistent storage is required, but this workshop uses temporary storage.

- You can see that some of the fields say "generated if empty". This is a feature of Templates in OpenShift. For now, be sure to use the following values in their respective fields:

- `Database Service Name` : `mongodb-nationalparks`

- `MongoDB Connection Username` : `mongodb`

- `MongoDB Connection Password` : `mongodb`

- `MongoDB Database Name`: `mongodb`

- `MongoDB Admin Password` : `mongodb`

These values are for workshop purposes and should be changed for a real deployment of a database. Click create, then go to secrets.

#### Configure Parameters and Check Replicas

In secrets, find the one with name `mongodb-ephemeral-parameters....`. Click the secret name and it will be use for **Parameters**. The secret can be used in other components, such as the `nationalparks` backend, to authenticate to the database. The connection and authentication information is stored in a secret in our project. Add it to the `nationalparks` backend. Click the **Add Secret to Workload** button. Select the `nationalparks` workload and click **Save**.

In Topology in the web console, move the mongodb into the workshop area in case it is not already there.

Add labels to the `mongodb-nationalparks` deployment:

`oc label dc/mongodb-nationalparks svc/mongodb-nationalparks app=workshop component=nationalparks role=database --overwrite`

Check the replica set and observe a new version of the nationalparks pod was deployed. The new version is due to the changes in secrets made in the previous steps.

`oc get rs`

The old versions are listed as Desired: 0, Current: 0.

#### Load data

Check data in the application

```sh
# Get back end URL
oc get routes | grep nationalparks
```

Visit the URL and add `/ws/data/all` to it (e.g. nationalparks-testuser.apps.cluster-sandbox.opentlc.com/ws/data/all). There will be no data, so let's load the data using the same URL except change the context to `ws/data/load` and then this message is displayed `Items inserted in database: 2893`.

This [`MongoDBConnection.java`](http://www.github.com/openshift-roadshow/nationalparks/blob/master/src/main/java/com/openshift/evg/roadshow/parks/db/MongoDBConnection.java#L44-l48) from the back end project shows how the database connection works. It took the environment variable to connect to the database. Environment variables, ConfigMaps, and secrets allow portability without changing application code.

#### Labels and Back end connectivity

**Labels** help the application understand routes and services.  If any of them have a **Label** that is `type=parksmap-backend`, the application checks endpoints to look for map data. Example is [`RouteWatcher.java`](https://github.com/openshift-roadshow/parksmap-web/blob/master/src/main/java/com/openshift/evg/roadshow/rest/RouteWatcher.java#L19) in the web front end.

Check labels on the `nationalparks` route and add a label so it is designated as a back end.

```sh
# Check existiong labels
oc describe route nationalparks
# Add label
oc label route nationalparks type=parksmap-backend
```

Use `oc get routes` to get the parksmap URL or check it in the web console. Check the front end parksmap URL and you will see points coming up on the map.

### Application Health: Add Readiness and Liveness Probes

A liveness probe checks if the container in which it is configured is still running. If the liveness probe fails, the kubelet kills thecontainer, which will be subjected to its restart policy. Set a liveness check by configuring the `template.spec.containers.livenessprobe` stanza of a pod configuration. A readiness probe determines if a container is ready to service requests. If the readiness probe fails acontainer, the endpoints controller ensures the container has its IP address removed from the endpoints of all services. A readiness probe canbe used to signal to the endpoints controller that even though a container is running, it should not receive any traffic from a proxy. Set areadiness check by configuring the `template.spec.containers.readinessprobe` stanza of a pod configuration.

See [Application Health](https://docs.openshift.com/container-platform/latest/applications/application-health.html) documentation for details.

In the web console: Topology > Select nationalparks > Actions: add/edit Health Checks

Add a Readiness Probe using Path: `/ws/healthz/`. Repeat for Liveness Probe with the same path. Leave all defaults as they are such as HTTP GET and port 8080. Check mark both probes and save the health checks page. The save will trigger new deployments of the pods.

### Continuous Integration and Pipelines

OpenShift Pipelines is a cloud-native, continuous integration and delivery (CI/CD) solution for building pipelines using [Tekton](https://tekton.dev/).

#### Deploy Pipeline

Deploy the pipeline stored in the code repository and verify the tasks and pipelines are present

```sh
oc create -f https://raw.githubusercontent.com/justintungonline/nationalparks/master/pipeline/nationalparks-pipeline-all-new.yaml -n <namespace>
oc get tasks -n <namespace>
oc get pipelines -n <namespace>
```

In the web console, view the pipelines under the Pipelines menu and see 4 Tasks defined:

- **git clone**: this is a `ClusterTask` that will clone source repository for nationalparks and store it to a `Workspace` `app-source` which will use the PVC created for it `app-source-workspace`
- **build-and-test**: will build and test our Java application using `maven` `ClusterTask`
- **build-image**: will build an image using a binary file as input in OpenShift. The build will use the .jar file that was created and a custom Task for it `s2i-java11-binary`
- **redeploy**: it will deploy the created image on OpenShift using the Deployment named `nationalparks` we created in the previous lab, using the custom Task `redeploy`

The pipeline will take parameters.

#### Run Pipeline

In the web console, pipeline detail, click Actions > Start:

- When prompted with parameters to add the Pipeline, add in **APP_GIT_URL** the `nationalparks` repository: `https://github.com/justintungonline/nationalparks.git`
- In **Workspaces**→ **app-source** select **PVC** from the list, then select **app-source-pvc**. This is the shared volume used by Pipeline Tasks in your Pipeline containing the source code and compiled artifacts.
- Leave Maven settings, it will use an **EmptyDir** volume for the maven cache, this can be extended also with a PVC to make subsequent Maven builds faster

Start the pipeline. In the web console, observe it running. Clicking on a task will show its logs. Verify the pipeline completes.

#### Automation on Code Changes

##### Pipeline Triggers

Git server repositories support the concept of web hooks --- calling to an external source via HTTP(S) when a change in the code repository happens. OpenShift provides an API endpoint that supports receiving hooks from remote systems in order to trigger builds. By pointing the code repository's hook at the OpenShift Pipelines resources, automated code/build/deploy pipelines can be achieved using:

- TriggerTemplate: a trigger template is a template for newly created resources. It supports parameters to create specific `PipelineResources` and `PipelineRuns`.
- TriggerBinding: validates events and extracts payload fields
- EventListener: connects `TriggerBindings` and `TriggerTemplates` into an addressable endpoint (the event sink). It uses the extracted event parameters from each TriggerBinding (and any supplied static parameters) to create the resources specified in the corresponding TriggerTemplate. It also optionally allows an external service to pre-process the event payload via the interceptor field.

Create a new Pod with a Route that can be used to setup a Webhook on a git server to trigger the automatic start of the Pipeline

```sh
oc create -f https://raw.githubusercontent.com/justintungonline/nationalparks/master/pipeline/nationalparks-triggers-all.yaml -n <namespace>

# Verify a new pod el-nationalparks is created as the EventListener
oc get pods
# Get the URL of the Event listener and copy it
oc get routes | grep el-nationalparks
```

Using the URL copied or get the URL from the web console Topology, the URL should something like:
<http://el-nationalparks-user.sandbox.opentlc.com/>

Make sure there is a `/` at the end.

##### Web Hooks

On the GitHub website, go to your `nationalparks` repository and add a new Webhook with Settings > Webhooks > Add.

- Paste the el-nationalparks URL into the webhook URL and change the `Content Type` to `application/json`.
- Leave the secret token field blank
- Choose `Just the push event`
- Add the webhook

New pushes to the `nationalparks` repository will now trigger a new build and deploy in OpenShift.

##### Trigger the Web Hook

In GitHub, go to your `nationalparks` repository and this file `src/main/java/com/openshift/evg/roadshow/parks/rest/BackendController.java`

Change line number 20:

```java
return new Backend("nationalparks","National Parks", new Coordinates("47.039304", "14.505178"), 4);
```

To

```java
return new Backend("nationalparks","Amazing National Parks", new Coordinates("47.039304", "14.505178"), 4);
```

Note the change in name with `Amazing`. Commit the change and in Openshift observe a new PipelineRun should be triggered. In the web console, click Pipeline in the left navigation menu then `nationalparks-pipeline`. You should see a new one running or run the following command to verify:

`oc get PipelineRun`

After the pipeline is completed, verify the change at the nationalparks URL similar to  `nationalparks-user.sandbox.opentlc.com/ws/info/`. Using the `/ws/info` context. The JSON returned will show:

```json
...
displayName "Amazing National Parks"
...
```

To see this `Amazing National Parks` text change in the parksmap’s legend itself, scale down the parksmap pod to 0, then back up to 1 to force the app to refresh its cache:

```sh
oc scale --replicas=0 deployment/parksmap && oc scale --replicas=1 deployment/parksmap
# Wait for parksmap to reload and verify it is running
oc get pods
```

The parksmap may take a few minutes to start up and load the new legend and data points again from the backend, then you can visit the parksmap URL.

#### Application Templates

Next is to deploy a map of Major League Baseball stadiums in the US by using a template. It is pre-configured to build the back end Java application, and deploy the MongoDB database. It also uses a **Hook** to call the `/ws/data/load` endpoint to cause the data to be loaded into the database from a JSON file in the source code repository.

Start using the template:

```sh
# Create the template in the project, a copy is stored in this workshop repository
oc create -f https://raw.githubusercontent.com/openshift-roadshow/mlbparks/master/ose3/application-template-eap.json
# Verify both the mlbparks and MongoDB (imported earlier) templates are available
oc get template
# Instantiate the template
oc new-app mlbparks -p APPLICATION_NAME=mlbparks
# View the mlbparks template
oc get template mlbparks -o yaml
```

The template can also use Maven for the build. This option is available if you provide the **MAVEN_MIRROR_URL** parameter with the location of a nexus repository like: `oc new-app mlbparks --name=mlbparks -p MAVEN_MIRROR_URL=http://nexus.labs.svc.cluster.local:8081/repository/maven-all-public`

This template completed the following in one command:

- Configure and start a build
  - Using the supplied Maven mirror URL (if you have specified the parameter)
  - From the supplied source code repository
- Configure and deploy MongoDB
  - Using auto-generated user, password, and database name
- Configure environment variables for the app to connect to the DB
- Create the correct services
- Label the app service with `type=parksmap-backend`

In the web console Topology view, put `mlbparks` and `mongodb-mlbparks` into the `workshop` application grouping if it isn't already there.

Another way to instantiate the template is using the web console Developer Perspective (**+Add** > **From Catalog** > search for `mlb`. Use result for `MLBparks` > Go through instantiation form. This step is not required as it was done on the command line already.

The template YAML can also be viewed/edited in the web console Developer Perspective: **Advanced → Search** in the left navigation > select **Template** > **mlbparks**.

### Binary Builds for Day to Day Development

Sometimes S2I is too slow for daily iterative development due to the full build and deploy. A faster option is to use `oc` command to use local code for S2I.

This pattern can be used to send up your working directory to the S2I builder to have it do the Maven build on OpenShift. Using the local directory would relieve you from having Maven or any of the Java toolchain on your local machine AND would also not require git commit with a push. Read more in the [official documentation](https://docs.openshift.com/container-platform/latest/dev_guide/dev_tutorials/binary_builds.html).

#### Using Binary Deployment

The next steps will clone the `mlbparks` source to the local environment, build it locally after a change, and use `oc` to use local binaries for the deployment.

```sh
# Clone source in a temporary directory
mkdir temp
cd temp
git clone https://github.com/justintungonline/mlbparks.git
cd mlbparks
# Build with Maven, get the output location for the .../mlbparks/target/ROOT.war, it will be needed for the oc command later
# Build application with Maven, it will take a while as Maven donwloads dependencies and then compiles the application
mvn package
```

In GitHub website or your editor of choice, change the source code in the `mlbparks` repository `src/main/java/com/openshift/evg/roadshow/rest/BackendController.java`. This is the REST endpoint that gives basic info on the service.

Please change line 23 to add a *AMAZING* in front of "MLB Parks" look like this:

```java
return new Backend("mlbparks", "AMAZING MLB Parks", new Coordinates("39.82", "-98.57"), 5);
```

Save the file and build the source again or edit it remotely and commit to the repository, then use `git pull`.

```sh
# Re-build the WAR file
mvn package
# Use oc to use the local WAR file
# The --follow is optional if you want to follow the build output in your terminal.
oc start-build bc/mlbparks --from-file=target/ROOT.war --follow
```

Using labels and a recreate deployment strategy, as soon as the new deployment finishes the map name will be updated. Under a recreate deployment strategy we first tear down the pod before doing our new deployment. When the pod is torn down, the parksmap service removes the MLBParks map from the layers. When it comes back up, the layer automatically gets added back with our new title. This would not have happened with a rolling deployment because rolling spins up the new version of the pod before it takes down the old one. Rolling strategy enables a zero-downtime deployment.

Follow the deployment in progress from Topology view, and when it is completed, you will see the new content at the `mlbparks` backend `/ws/info/` URL.

## Configuration Management

### How to pass things into a container?

Options:

- Mount a volume into container
- Set environment variables, using yaml, example is Java processes can use system variables
- ConfigMaps can configure these items
- In web console > pod detail view > Environment tab. Choose environment information and secrets. Database creation steps above are an example of passing variables to the container.

Pod yaml example:

```yaml
...
spec:
    containters:
        - name: parksmap
        ...
        env:
            - name: testvariable
              value: testvalue
...
```

#### [ConfigMaps](https://docs.openshift.com/container-platform/4.6/authentication/configmaps.html)

 It contains key value pairs to store configurations as read only. It is like properties.

Example `yaml`

```yaml
kind: ConfigMap
...
data:
    example.property.1: hello
    settings.json: |-
        { 
            # json data with key value pairs
        }
...
```

#### Secrets

Secrets are encoded. Example `yaml`

```yaml
kind: Secret
...
data:
    database-name: aasd2tgs= 
    database-password: dXNgfdgepg==
...
```

### How does the front end find the back end?

The front end can query the Kubernetes API to find backends. Finding is done in the application code such as this [`RouteWatcher.java`](https://github.com/openshift-roadshow/parksmap-web/blob/master/src/main/java/com/openshift/evg/roadshow/rest/RouteWatcher.java)

## Further Resources

- **[OpenShift Interactive Learning Portal](https://learn.openshift.com/)** - An online interactive learning environment where you can run through various scenarios related to using OpenShift.

The online interactive learning environment is always available so you can continue to work on those exercises even after the workshop is over.

Below you will find further resources for learning about OpenShift and running OpenShift on your own computer, as well as details about OpenShift Online or other OpenShift related products and services.

- **[OpenShift Documentation](https://docs.openshift.com)** - The landing page for OpenShift documentation.

- **[OpenShift Resources on developers.redhat.com](https://developers.redhat.com/openshift/)** - A collection of resources for developers who are building and deploying applications on OpenShift.

- **[CodeReady Containers](https://developers.redhat.com/products/codeready-containers/overview)** - A tool which can be used to install a local OpenShift cluster on your own computer, running in a virtual machine.

- **[OpenShift Online](https://manage.openshift.com/)** - A shared public hosting environment for running your applications using OpenShift.

- **[OpenShift Dedicated](https://www.openshift.com/dedicated)** - A dedicated hosting environment for running your applications, managed and supported for you by Red Hat.

- **[OpenShift Container Platform](https://www.openshift.com/)** - The Red Hat supported OpenShift product for installation on premise or in hosted cloud environments.

- **[OpenShift Commons](https://commons.openshift.org)** - A community for users, partners, customers, and contributors to come together to collaborate and work together on OpenShift.

The following free online eBooks are also available for download related to OpenShift.

- **[OpenShift for Developers](https://www.openshift.com/for-developers/)**

- **[DevOps with OpenShift](https://www.openshift.com/devops-with-openshift/)**

- **[Deploying with OpenShift](https://www.openshift.com/deploying-to-openshift/)**
