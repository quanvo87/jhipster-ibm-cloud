# Deploy a JHipster BFF to IBM Cloud

[JHipster](http://www.jhipster.tech/) "is a development platform to generate, develop and deploy Spring Boot + Angular Web applications and Spring microservices."

In this blog post, we'll generate a JHipster backend for frontend (BFF), and then deploy it to Kubernetes on IBM Cloud.

### Requirements

#### JHipster

Visit [JHipster](http://www.jhipster.tech/) for requirements and installation instructions (scroll to their Quick Start section).

#### IBM Cloud

- Install the tools:
  - IBM Cloud CLI
  - IBM Cloud Container Service plug-in
  - Kubernetes CLI
  - IBM Cloud Container Registry plug-in
  - Docker CLI

*Instructions can be found in Lesson 1 of these [docs](https://console.stage1.bluemix.net/docs/containers/cs_tutorials.html#cs_cluster_tutorial).*

- Create a Kubernetes cluster

*Instructions can be found in Lesson 2 of the docs above (you can skip steps 5 and 6).*

### Generate Your JHipster Microservice

- We are going to quickly generate a JHipster BFF. For a more detailed walkthrough, see these tutorials:
  - [https://github.com/oktadeveloper/jhipster-microservices-example/blob/master/TUTORIAL.md](https://github.com/oktadeveloper/jhipster-microservices-example/blob/master/TUTORIAL.md)
  - [http://www.baeldung.com/jhipster-microservices](http://www.baeldung.com/jhipster-microservices)
- Open your terminal, and create a directory for your project, for example `jhipster-bff`

#### First we need to clone the jhipster-registry:

- `cd` into the directory you just created
- Run `git clone git@github.com:jhipster/jhipster-registry.git`

#### Next we can create our microservice:

- Go back to your project's main folder and make another directory called `microservice` and `cd` into it
- Run the `jhipster` command. This will begin the generator.
- For the prompts, answer similar to this:
  - Type of application: microservice application
  - Basename: microservice
  - Port: 8081
  - Default Java package name: com.example.microservice
  - Service discovery server: JHipster Registry
  - Type of authentication: JWT
  - Type of database: SQL
  - Production database: MySQL
  - Development database: H2 with disk-based persistence
  - Hibernate 2nd level cache: Yes, with HazelCast
  - Maven or Gradle: Maven
  - Other technologies: none
  - Internationalization support: no
  - Testing frameworks: none
  - Install other generators: no

#### Once that finishes generating, we can create an entity. We'll call it `product`:

- Run `jhipster entity product`. This starts the entity generator.
- For the prompts, answer similar to this:
  - Add a field: yes
  - Name of field: name
  - Type of field: String
  - Validation rules: no
  - Add a field: yes
  - Name of field: price
  - Type of field: Double
  - Validation rules: no
  - Add a field: no
  - Add a relationship to another entity: no
  - Use Data Transfer Object: No, use the entity directly
  - Use separate service class for business logic: no
  - Pagination: no
- It will ask you if you want to overwrite files, enter `a` to say yes to all
- You have just created your `product` entity!

#### Next we'll create our gateway:

- Go back to your project's main folder and make another directory called `gateway` and `cd` into it
- Run the `jhipster` command. This will begin the generator.
- For the prompts, answer similar to this:
  - Type of application: microservice gateway
  - Basename: gateway
  - Port: 8080
  - Default Java package name: com.example.gateway
  - Service discovery server: JHipster Registry
  - Type of authentication: JWT
  - Type of database: SQL
  - Production database: MySQL
  - Development database: H2 with disk-based persistence
  - Maven or Gradle Maven
  - Other technologies: none
  - Client framework: Angular 4
  - SASS support: yes
  - Internationalization: no
  - Testing frameworks: none
  - Install other generators? no

#### Once that is done, we can create the UI for our `product` entity:

- Run `jhipster entity product`
- For the prompts, answer similar to this:
  - Generate from an existing microservice: yes
  - Path: ../microservice
  - Update the entity: Yes, re generate the entity
- It will ask to overwrite files, enter `a` to say yes to all
- The UI for our `product` entity is now generated!

#### Now we can test the our BFF locally before deploying to Kubernetes on IBM Cloud:

First, start the registry:
- Go back to the main directory of our project. Recall the registry we cloned at the beginning of this post.
- `cd` into `jhipster-registry`
- Start the application: run `./mvnw`

Start the microservice:
- In another terminal window, go to the `microservice` directory and run `./mvnw`
  - *The registry must be up and running for this to work*

Start the gateway:
- In another terminal window, go to the `gateway` directory and run `./mvnw`
  - *The microservice must be up and running for this to work*

Once everything is running, you can visit your gateway at [http://localhost:8080/](http://localhost:8080/). If everything was successful, you should be able to:
- Log in with username `admin` and password `admin`
- Add a `product` entity

![jhipster](https://developer.ibm.com/java/wp-content/uploads/sites/136/2018/01/jhipster.png)

Now we're ready to deploy our application!

### Deploy To Kubernetes on IBM Cloud

We will use the `jhipster kubernetes` sub generator to help us deploy to IBM Cloud.

- Make sure you have your environment set up as per the requirements at the top of this post
- Start Docker

#### Create Docker images for our `gateway` and `microservice`:

- In the `gateway` directory, run `./mvnw package -Pprod dockerfile:build`
- Repeat the command in the `microservice` directory

#### Now we can run the Kubernetes sub generator:

- In the main directory of your project, create a folder called `kubernetes`
- `cd` into it and run `jhipster kubernetes`
- For the prompts, answer something like:
  - Type of application: microservice
  - Root directory: ../
  - Which applications to include: select `gateway` and `microservice`
  - Password: admin
  - Kubernetes namespace: default
  - Base Docker repository name: put any value, we will have to edit this later
  - Command to push Docker image: docker push
  - Use JHipster Console: yes
  - Export services to Prometheus: no
  - Kubernetes service type: NodePort

Our Kubernetes config files have now been generated. We will need to do some further configuration in order to deploy our app to IBM Cloud.

#### Log in to IBM Cloud:

- Run `bx login` or `bx login --sso`
- View your clusters: `bx cs clusters`
- Download config info on a cluster: `bx cs cluster-config {cluster-name}`
  - This returns an export statement. Copy and paste it, and run it in your terminal to export environment variables to start using Kubernetes.
  - It will look something like: `export KUBECONFIG=/Users/quanvo/.bluemix/plugins/container-service/clusters/cluster/kube-config-hou02-cluster.yml`

#### Push Docker images to your container registry:

For your gateway and each microservice, you need to push a Docker image to a container registry. You can use your IBM Cloud container registry.

- Log in to the container registry: `bx cr login`
- Make a container registry namespace if needed: `bx cr namespace-add {your-namespace}`
- You can view your namespaces with `bx cr namespaces`

- For each app (your gateway, each microservice), repeat the following steps:
  - `cd` into the app directory
  - Build the image: `docker build -t registry.ng.bluemix.net/{your-namespace}/{app}:1.0 target`
    - e.g. if you named your gateway `gateway`, put in `gateway` for `{app}`
  - Push the image to your IBM Cloud container registry: `docker push registry.ng.bluemix.net/{your-namespace}/{app}:1.0`
  - You can verify the image was pushed with `bx cr images`

#### Create Kubernetes deployments and services:

`cd` into the folder you ran the `jhipster kubernetes` command in. Let's call this your Kubernetes folder.

First create a deployment and service for your registry: `kubectl apply -f registry`

Inside your Kubernetes folder, you'll notice additional folders for each microservice you chose were generated. For each of these folders, open the `{app}-deployment.yml` file. e.g. for your gateway, the file should be called something like `gateway-deployment.yml`. In the yml files, make these changes:

- Under spec -> containers -> image, update to the correct image on your container registry
  - e.g. `registry.ng.bluemix.net/{your-namespace}/{app}`
  - Don't forget the tag, e.g. `microservice:1.0`
- Remove the `resources`, `readinessProbe`, and `livenessProbe` sections
- Save
- Back in your Kubernetes folder, create the deployment and service: `kubectl apply -f {app-folder}`
  - e.g. for your gateway, something like `kubectl apply -f gateway`
- You can confirm the deployment and service is running correctly by checking the Kubernetes dashboard. run `kubectl proxy &` and open a browser to [localhost:8001/ui](http://localhost:8001/ui)

#### Access your app:

- View your clusters: `bx cs clusters`
- Get the public IP of your node: `bx cs workers {cluster-name}`
- Get the port that the gateway service is exposed on: `kubectl describe service {gateway-service-name}`
  - Your gateway service name is probably `gateway`. You can check this in the Kubernetes dashboard.
  - If your gateway service type is `NodePort`, the exposed port will be on this line: `NodePort: web 30759/TCP`
- Access your app at `{node-public-ip}:{port}`
  - e.g. [184.172.214.179:30579]()
- You should see an app similar to the one you ran locally
- If everything was successful, you should again be able to log in with username `admin` and password `admin`, and create a `product`

You have just generated a JHipster BFF and deployed it to Kubernetes on IBM Cloud!
