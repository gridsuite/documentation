# GridSuite documentation

## Architecture

Here is a micro-service architecture diagram. Yellow boxes are micro-services, orange boxes are user interfaces and everything is blue are external services (like databases, message broker, etc). Arrows represent http, websockets or AMPQ connections. We use 3 different databases: PostGresSQL which is the default solution, Cassandra DB for micro-services requiring high throughput and good scaling and ElasticSearch for data indexing. In addition to databases, we also use file system and FTP storages. Most of the micro-services communication rely on synchronous http request (REST APIs), but we also have asynchronous communication thought a RabbitMQ message broker. 



![gridsuite_architecture.drawio](diagrams/gridsuite_architecture.svg)

## Micro-services description

### Network store server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-network-store
- Storage: Cassandra DB
- Connected to Message broker: no
- Other services dependencies: none

This service is responsible for storing network in [IIDM](https://www.powsybl.org/pages/documentation/grid/model/) format. Each of the network elements is stored in its own table indexed by its ID and voltage level ID allowing 3 levels of queries for a given network: all elements, an element from its ID, all elements belonging to a substation voltage level. This web services is not intended to be accessed directly though its REST API but using the Java Client. The java Client is an alternative implementation of the [IIDM API](https://github.com/powsybl/powsybl-core/tree/main/iidm/iidm-api) (alternative regarding the original in memory one from PowSyBl) and can be found in the network-store-iidm-impl module of the repository. An integration of the Java client into Spring framework is also available int he network-store-client module.

### Case server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-case
- Storage: file system and ElasticSearch
- Connected to message broker: yes, message producer
- Other services dependencies: no

This service is responsible for storing case in any of PowSyBl supported formats (CGMES, UCTE, XIIDM, etc). Cases are just files and are store in the current version on a file system. The goal is to migrate to an Object Storage solution (like minIO) and to access it via a standard API for instance Amazon S3 API. Cases can be stored in 2 differents spaces: one "public" and one "private". Cases from the public space are accessible to any other services and all public cases can be listed and loaded without any restriction. Cases from private space are assigned to a unique identifier (UUID) when loaded to the case server and can only be accessed knowing this unique identifier. There is no way to list private cases. When a new case is uploaded to the server, it is indexed into an ElasticSearch database so that we can query the case server (using the REST API) for searching a case matching some provided criteria. In the current version, we only support simple index attribute coming from the case file name (like date, TSO, country, etc). In a future version, we will add more index attributes coming from the case content (like network element IDs, consumption or generation level, etc). Also, a message is sent to the broker each time a case is uploaded, so that other services can trigger some processing as soon as a new case that meet some criteria enter into the system.

### Report server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/report-server
- Storage: PostGresSQL
- Other services dependencies: no

This service is responsible for storing function logs. Functional logs unlike technical logs (managed by Slf4j/Logback and EFK stack) are designed to be presented to a user.  Is is often less verbose that technical logs and contains meta-data associated to each log (for instance numerical variable values and their unit line MW or A). In the current version of the service, there is very few query capabilities but the aim is to be able to select logs from a time interval, from key types, etc. Each log is associated by a unique identifier (UUID) to a container which can be for instance a study node or a merge from the merge orchestrator server or any kind of application that need to have associated functional logs.  Functional logs Java modeling is handled by [PowSyBl reporter API](https://github.com/powsybl/powsybl-core/tree/main/commons/src/main/java/com/powsybl/commons/reporter) and is used in all micro-services that need to generate logs JSON document. Many of PowSyBl underlying features are capable of generating functional logs by just passing a  `Reporter` in parameter. On front-end side a [generic ReactJs component](https://www.npmjs.com/package/@gridsuite/commons-ui) is available to display functional logs. 

### Network conversion server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-network-conversion-server
- Storage: none
- Connected to message broker: no
- Other services dependencies: network store server, case server and report server

This service is responsible for importing cases to and exporting cases from the network store. Supported case importers and exporters are [the ones available from PowSyBl](https://www.powsybl.org/pages/documentation/index.html#grid-formats). The case is provided to the API either by providing the full case file or by just referring to a case in the case server. During the case conversion, functional logs are generated.  Once conversion has been done and network fully stored to network store server, functional logs are stored to the report server for later analysis, search or UI display.

### Single line diagram server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-single-line-diagram-server
- Storage: none
- Connected to message broker: no
- Other services dependencies: network store server

This service is responsible for generating single line diagrams either for a voltage level or for a full substation from a network stored into network store server. Single line diagram generation feature is provided by a [PowSyBl component](https://github.com/powsybl/powsybl-single-line-diagram). A single line diagram is a SVG file and additional metadata (mainly to keep links with network elements).

### Geo data server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/geo-data
- Storage: Cassandra DB
- Connected to message broker: no
- Other services dependencies: network store server

This service is responsible for storing network related geographical data. There are two kind of geographical data: substation ones and line ones. Substation geographical data are just one GPS (WGS84) coordinate associated to a substation ID. Line geographical data are a list of GPS coordinates corresponding to the position of all line pylons (so substations position at both side are not included). When data is query for a specific network, missing data are automatically created. For substations, missing coordinates are computed from neighbor substations using a centroid algorithm. For lines, coordinates are never required by front-end (a straight line is drawn in case of missing data) and different fixing algorithms are used in case of inconsistent data: for instance when the list of pylon coordinates do not allow creation of a polyline (because of missing segments). 

### CGMES GL server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-cgmes-gl
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, geo data server

This service is responsible of converting CGMES GL (Graphical Layout) profile to GPS coordinates uploaded to geo data server. 

### ODRE server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/odre-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, geo data server

This services is responsible for converting substations and lines GPS coordinates coming from [Open Data Réseau Energies](https://opendata.reseaux-energies.fr/pages/accueil/?flg=fr) system (ODRE) and upload it geo data server. ODRE is a web site and a web service where RTE equipments GPS positions  have been published in open data. 

### Network map server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/network-map-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server

This service is responsible for extracting/filtering/reshaping network data from network store, so that it can be used on front-end side to feed to the network map, the spreadsheet and many other components that need to display network data.

### Network modification server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/network-map-server
- Storage: PostGresSQL
- Connected to message broker: no
- Other services dependencies: network store server, report server

This service is responsible for storing and applying modification to a network of the network store server. This is one of the most important service of the application as it implements many businness logic for modifying the network. There is many kind of modification supported by this service like network element creation (substation, voltage level, generator, load, etc), network element removal, switch opening or closing, line or transformer triggering or locking out, network element elementary modification (like a target value modification, or a tap position modification). Only a very few number of expected possible modifications have yet been implemented and many more will come in the future. Network modifications have a unique ID (UUID) and could be associated to a group which have itself a unique ID (UUID) and this is how a set of modifications is referenced in the database of other web services. Depending of the use case a network modification can be only stored, can be stored and applied to a network  (given a network ID in the network store) or just selected from database and applied to a network. When a modification is applied to a network functional logs are created to give more informations to the users and sent to the report server for later use. 

### Load flow server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/loadflow-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, report server

This service is reponsible for running a [loadflow](https://en.wikipedia.org/wiki/Power-flow_study) on a network of the network store server. It relies on [PowSyBl loadflow API](https://github.com/powsybl/powsybl-core/tree/main/loadflow/loadflow-api), a generic, implementation agnostic API, so that as soon as a new implementation is available, it can be integrated to this service without adding any new code. In the current version, two loadflow implementation are integrated: [Hades2](https://rte-france.github.io/hades2/) which is a free software made by RTE and [OpenLoadFlow](https://github.com/powsybl/powsybl-open-loadflow) a loadflow fully implemented in PowSyBl. A third one, [DynaFlow](https://github.com/dynawo/dynaflow-launcher) is going to be integrated soon. When a loadflow has ran on a network, we get results like status, metrics, functional logs and above all, state variables (voltages, flows) updated to the network store.  

### Action server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/actions-server
- Storage: PostGresSQL
- Connected to message broker: no
- Other services dependencies: network store server

This service is resposible for storing and instanciating contingency lists. Contingencies are used by security analysis to describe network elements that we want to simulate the loss of. We support two main types of contingency list:  form based and script based. A form based contingency list is described by an element type (generator, line, etc) and some filtering criteria (country, nomina voltage, etc), so it needs to be instanciated with a specific network to be used by a security analysis. A script based contingency list is a [Groovy](https://groovy-lang.org) script where the network is exposed a global variable so that the user can write in Groovy the logic to generate contingency from network elements.

### Filter server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/filter-server
- Storage: PostGresSQL
- Connected to message broker:
- Other services dependencies: network store server

This service is responsible for storing and instanciating filters. Filters are just set of network elements and are useful in many places in the application: to help configuring network modifications, to filter data in the UIs, etc. The overall design of filters is very close to contingency lists: there is form based and Groovy script based filters.

### Security analysis server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/security-analysis-server
- Storage: Cassandra DB
- Connected to message broker: consumer and producer
- Other services dependencies: network store server



### Directory server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/directory-server
- Storage: PostGresSQL
- Connected to message broker: producer
- Other services dependencies: 

### Dynamic simulation server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/dynamic-simulation-server
- Storage: Cassandra DB
- Connected to message broker: producer and consumer
- Other services dependencies: network store server

### Dynamic mapping server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/dynamic-mapping-server
- Storage: PostGresSQL
- Connected to message broker: no
- Other services dependencies: network store server

### Study server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/study-server
- Storage: PostGresSQL
- Connected to message broker: producer and consumer
- Other services dependencies: network store server, case server, single line diagram server, network conversion server, geo data server, network map server, network modification server, loadflow server, actions server, security analysis server, report server

### Explore server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/explore-server
- Storage: no
- Connected to message broker:
- Other services dependencies: directory server, study server, actions server, filter server

### Config server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/config-server
- Storage: PostGresSQL
- Connected to message broker: producer
- Other services dependencies:

This service is responsible for storing parameters coming from GridSuite front-ends. A parameter is a key value (String, String) type data associated to a user and an application (just a String like 'study' or 'merge'). Thanks to this service front-ends can saved their states (like options in parameters dialog) and users are able to keep same UI configuration whatever browser or machine is used to connect to GridSuite.



###  Balances adjustment server

TODO

### Merge orchestrator server

TODO

### Case validation server

TODO

### CGMES boundary server

TODO



### Notification server

### Directory notification server

### Config notification server

### Merge notification server



### Gateway

- Kind: Gateway
- Source repository: https://github.com/gridsuite/gateway
- Storage: no
- Connected to message broker: no
- Other services dependencies: study server, action server, filter server, directory server, case server

This is the only entry point to the back-end. So front-ends can only send requests to the gateway. Request are then routed to other micro services. This gateway is mainly responsible from implementing secury features: https and user access right verification.

### Case import job

### CGMES boundary import job

### CGMES assembling job



## Front-ends description:

###  Grid Explore

### Grid Study

### Grid Merge

### Grid Dyna

### Grid Geo
