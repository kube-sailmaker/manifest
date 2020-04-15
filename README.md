# Sailmaker manifest
Document repository for apps, specs, environments, resources, mixins

### Inspiration
Ship manifest - A document listing the cargo, passengers, and crew of a ship for the use of customs and other officials

### Resources by Role
-----------------
The manifest project contains a sample project layout which highlights the use of various roles to foster
collaboration between various members in a team.

We have also grouped the manifests into two namespaces - user, provider.

Infrastructure, Resource and Mixins are not as frequently changed and are provided. Apps and Releases change from time to time and belong in user namespace. 
The manifests files in this project resemble Kubernetes Manifest files. There are many DSLs and wordings, we definitely
could do with one less DSL. It is the project's aim to have these manifests registered as CRDs in Kubernetes.

Also note, even though we are using this manifest to deploy to Kubernetes, it is agnostic of Kubernetes runtime and could
easily be used to deploy on non-kubernetes environment.

### The Where, What & How
- infrastructure - where are the things I need at runtime?
- resources - what are the logical names of these things?
- requirements - how should I be deploying in an environment?
- apps - what are we working with?
- releases - how are we shipping/consuming the apps? 

In summary, we are deploying ```apps``` having specific ```mixins``` in many different ```releases``` by binding ```resources``` 
hosted in various ```infrastructures```.

#### Product Client
The role of a Product Client can be played by Front End teams or Product Owners.

```
    .
    └── releases                    # Directory to bundle microserive by enviroment first and grouped by namespace
        └── test                    # Test environment
            └── account_ns.yaml     # Sample test environment account ns bundle
```

#### Platform Owners
This role can be played by devops/SRE type of individuals who are responsible to own and define cross cutting concerns
and best practices around deployment strategies.
  
```
    .
    └── mixins                    # Overall language specific settigs
        └── java
        └── node 
```


#### Microservices Owners
The owner of the microservices knows their application. The manifests created by them could potentially be used in a 
different setup by a product team. In some scenarios, the microservices owner is a publisher in an internal marketplace 
where others can use the recipes in various product deployments. 

```
    .
    └── apps                        # Director for deployment configurations
        └── adimono.yaml            # Sample microserive
```

#### Resource Owners
Resource Owners are internal or external members who provide various infrastructure components/services for upstream clients.

```
    .
    └── resources                   # Directory for resources configs
        └── elasticsearch_user.yaml # Elasticsearch Resource Binding
    └── infrastructure              # Directory for infrastructure configuration
        └── cassandra-cluster-a.yaml #infrastructure definition for a cassandra cluster
        └── postgrees-db1.yaml      #infrastructure definition for a postgres cluster 
``` 


### Collaborations
This table describes the level of collaboration that is possible under this resource layout model.

|  Task | Owner  | Collaboration  | 
|---|---|---|
| Infrastructure  |  Data Platform |  Deployment Tooling /Data Platform Owners |
| Resources  | Data Platform  | Data Platform/Microservices/Data Engineering  |
| Apps  | Developer  | Product Client/Developer/SRE/DevOps  |
| Mixins        | DevOps     | DevOps/Developer |
| Release        | Product Client | DevOps/Product Engineer |

### Manifests Up-Close

#### An Infrastructure Manifest
An Infrastructure manifest has a name and have various template entries. Providers can one or more entry templates to share infrastructure attributes 
like the contact points and any infrastructure flags.  

```
kind: Infrastructure
metadata:
  name: postgres-db1
spec:
  template:
    - name: test
      attributes:
        contact_points: jdbc:postgres://ps-1.test.local.cluster:5432
        ssl: true
        availability_zone: "ap-southeast-2a"

    - name: alpha
      attributes:
        contact_points: jdbc:postgres://ps-1.alpha.local.cluster:5432
        ssl: true
        availability_zone: "ap-southeast-2b"

    - name: prod
      attributes:
        contact_points: jdbc:postgres://ps-1.prod.local.cluster:5432
        ssl: true
```

#### A Resource Manifest
A resource manifest helps expose infrastructure to applications using logical names. One or many resources can be created
from an infrastructure. It is also possible to create one resource from many infrastructures but care must be taken when doing 
this as it can potentially create conflicts.

```
apiVersion: v1
kind: Resource
metadata:
  name: cassandra-a
spec:
  template:
    - name: test1
      user_keyspace: tst_user
      account_keyspace: tst_account
      infrastructure:
        - cassandra-a/test

    - name: sit
      user_keyspace: sit_user
      account_keyspace: sit_account
      infrastructure:
        - cassandra-a/test

    - name: alpha
      user_keyspace: user
      account_keyspace: account
      infrastructure:
        - cassandra-a/alpha

```


#### Mixins Manifest
Mixins are reusable elements which can be used by one or many application deployments. Multiple mixins can be 
requested. The value of the ```salience``` attribute determines the precedence.  
```
- name: java-default
  cpu: c1
  memory: m1
  replicas: 1
  resource-limit-strategy: "half" #half, exact, none
  env:
    JAVA_OPTS: "-Xms256m -Xmx256m -Dlog4j.configurationFile=/opt/app/log/log4j2.xml"

- name: tiny
  cpu: c0
  memory: m0
  resource-limit-strategy: "exact"
  replicas: 1
```

If an app were to use the above two mixins, the cpu allocated to the resource would be whatever c0 translates to.
A range of cpu and memory values can be configured per environment.  

#### Application Manifest
Application manifest provides specification for what the app should be provided with at runtime. Custom annotations
can be provided to tag the application with ownership details and various other auxiliary attributes.
Resource dependencies can be specified in the resources array. Similarly, capabilities can be requested through capabilities
array. Capabilities can be things like enrichment request for prometheus scraping to vault access or read access to Kubernetes Api.

Templates provide instance specific overrides and mimics recipe in structure. Configuration specified in template take precedence
over ones inherited from mixins. In the example below, prod template will use cpu value of c1 whereas other templates will inherit 
c0 because of the dependency on tiny trait. 
  
```
name: minimal-app

annotations:
  owner: team1
  email: team@localhost

resources: 
    - name: elasticsearch-user
capabilities:
    - name: prometheus
    - name: vault

mixins:
  - tiny

template:
  - name: test
  - name: lab
  - name: prod
    config:
      cpu: c1
      memory: m1
```


#### Release Manifest
The release manifest gets converted to a custom resource definition so its progress can be tracked. Each release resource
can have one or many apps. An alias can be provided if an app is to be deployed multiple times or simply under another name. 

```
apps:
  - name: adimono
    version: 1.0

  - name: forcg
    version: 1.0

  - name: forcg
    version: 1.0
    alias: forcg-2
```

