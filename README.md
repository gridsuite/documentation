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
- Use PowSyBl libraries: yes

This service is responsible for storing network in [IIDM](https://www.powsybl.org/pages/documentation/grid/model/) format. Each of the network elements is stored in its own table indexed by its ID and voltage level ID allowing 3 levels of queries for a given network: all elements, an element from its ID, all elements belonging to a substation voltage level. This web services is not intended to be accessed directly though its REST API but using the Java Client. The java Client is an alternative implementation of the [IIDM API](https://github.com/powsybl/powsybl-core/tree/main/iidm/iidm-api) (alternative regarding the original in memory one from PowSyBl) and can be found in the network-store-iidm-impl module of the repository. An integration of the Java client into Spring framework is also available int he network-store-client module.

### Case server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-case
- Storage: file system and ElasticSearch
- Connected to message broker: yes, message producer
- Other services dependencies: no
- Use PowSyBl libraries: yes

This service is responsible for storing case in any of PowSyBl supported formats (CGMES, UCTE, XIIDM, etc). Cases are just files and are store in the current version on a file system. The goal is to migrate to an Object Storage solution (like minIO) and to access it via a standard API for instance Amazon S3 API. Cases can be stored in 2 differents spaces: one "public" and one "private". Cases from the public space are accessible to any other services and all public cases can be listed and loaded without any restriction. Cases from private space are assigned to a unique identifier (UUID) when loaded to the case server and can only be accessed knowing this unique identifier. There is no way to list private cases. When a new case is uploaded to the server, it is indexed into an ElasticSearch database so that we can query the case server (using the REST API) for searching a case matching some provided criteria. In the current version, we only support simple index attribute coming from the case file name (like date, TSO, country, etc). In a future version, we will add more index attributes coming from the case content (like network element IDs, consumption or generation level, etc). Also, a message is sent to the broker each time a case is uploaded, so that other services can trigger some processing as soon as a new case that meet some criteria enter into the system.

### Report server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/report-server
- Storage: PostGresSQL
- Other services dependencies: no
- Use PowSyBl libraries: yes

This service is responsible for storing function logs. Functional logs unlike technical logs (managed by Slf4j/Logback and EFK stack) are designed to be presented to a user.  Is is often less verbose that technical logs and contains meta-data associated to each log (for instance numerical variable values and their unit line MW or A). In the current version of the service, there is very few query capabilities but the aim is to be able to select logs from a time interval, from key types, etc. Each log is associated by a unique identifier (UUID) to a container which can be for instance a study node or a merge from the merge orchestrator server or any kind of application that need to have associated functional logs.  Functional logs Java modeling is handled by [PowSyBl reporter API](https://github.com/powsybl/powsybl-core/tree/main/commons/src/main/java/com/powsybl/commons/reporter) and is used in all micro-services that need to generate logs JSON document. Many of PowSyBl underlying features are capable of generating functional logs by just passing a  `Reporter` in parameter. On front-end side a [generic ReactJs component](https://www.npmjs.com/package/@gridsuite/commons-ui) is available to display functional logs. 

### Network conversion server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-network-conversion-server
- Storage: none
- Connected to message broker: no
- Other services dependencies: network store server, case server and report server
- Use PowSyBl libraries: yes

This service is responsible for importing cases to and exporting cases from the network store. Supported case importers and exporters are [the ones available from PowSyBl](https://www.powsybl.org/pages/documentation/index.html#grid-formats). The case is provided to the API either by providing the full case file or by just referring to a case in the case server. During the case conversion, functional logs are generated.  Once conversion has been done and network fully stored to network store server, functional logs are stored to the report server for later analysis, search or UI display.

### Single line diagram server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-single-line-diagram-server
- Storage: none
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for generating single line diagrams either for a voltage level or for a full substation from a network stored into network store server. Single line diagram generation feature is provided by a [PowSyBl component](https://github.com/powsybl/powsybl-single-line-diagram). A single line diagram is a SVG file and additional metadata (mainly to keep links with network elements).

### Geo data server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/geo-data
- Storage: Cassandra DB
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for storing network related geographical data. There are two kind of geographical data: substation ones and line ones. Substation geographical data are just one GPS (WGS84) coordinate associated to a substation ID. Line geographical data are a list of GPS coordinates corresponding to the position of all line pylons (so substations position at both side are not included). When data is query for a specific network, missing data are automatically created. For substations, missing coordinates are computed from neighbor substations using a centroid algorithm. For lines, coordinates are never required by front-end (a straight line is drawn in case of missing data) and different fixing algorithms are used in case of inconsistent data: for instance when the list of pylon coordinates do not allow creation of a polyline (because of missing segments). 

### CGMES GL server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-cgmes-gl
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, geo data server
- Use PowSyBl libraries: yes

This service is responsible of converting CGMES GL (Graphical Layout) profile to GPS coordinates uploaded to geo data server. 

### ODRE server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/odre-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, geo data server
- Use PowSyBl libraries: no

This services is responsible for converting substations and lines GPS coordinates coming from [Open Data Réseau Energies](https://opendata.reseaux-energies.fr/pages/accueil/?flg=fr) system (ODRE) and upload it geo data server. ODRE is a web site and a web service where RTE equipments GPS positions  have been published in open data. 

### Network map server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/network-map-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for extracting/filtering/reshaping network data from network store, so that it can be used on front-end side to feed to the network map, the spreadsheet and many other components that need to display network data.

### Network modification server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/network-map-server
- Storage: PostGresSQL
- Connected to message broker: no
- Other services dependencies: network store server, report server
- Use PowSyBl libraries: yes

This service is responsible for storing and applying modification to a network of the network store server. This is one of the most important service of the application as it implements many businness logic for modifying the network. There is many kind of modification supported by this service like network element creation (substation, voltage level, generator, load, etc), network element removal, switch opening or closing, line or transformer triggering or locking out, network element elementary modification (like a target value modification, or a tap position modification). Only a very few number of expected possible modifications have yet been implemented and many more will come in the future. Network modifications have a unique ID (UUID) and could be associated to a group which have itself a unique ID (UUID) and this is how a set of modifications is referenced in the database of other web services. Depending of the use case a network modification can be only stored, can be stored and applied to a network  (given a network ID in the network store) or just selected from database and applied to a network. When a modification is applied to a network functional logs are created to give more informations to the users and sent to the report server for later use. 

### Load flow server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/loadflow-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, report server
- Use PowSyBl libraries: yes

This service is reponsible for running a [loadflow](https://en.wikipedia.org/wiki/Power-flow_study) on a network of the network store server. It relies on [PowSyBl loadflow API](https://github.com/powsybl/powsybl-core/tree/main/loadflow/loadflow-api), a generic, implementation agnostic API, so that as soon as a new implementation is available, it can be integrated to this service without adding any new code. In the current version, two loadflow implementation are integrated: [Hades2](https://rte-france.github.io/hades2/) which is a free software made by RTE and [OpenLoadFlow](https://github.com/powsybl/powsybl-open-loadflow) a loadflow fully implemented in PowSyBl. A third one, [DynaFlow](https://github.com/dynawo/dynaflow-launcher) is going to be integrated soon. When a loadflow has ran on a network, we get results like status, metrics, functional logs and above all, state variables (voltages, flows) updated to the network store.  

### Action server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/actions-server
- Storage: PostGresSQL
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is resposible for storing and instanciating contingency lists. Contingencies are used by security analysis to describe network elements that we want to simulate the loss of. We support two main types of contingency list:  form based and script based. A form based contingency list is described by an element type (generator, line, etc) and some filtering criteria (country, nomina voltage, etc), so it needs to be instanciated with a specific network to be used by a security analysis. A script based contingency list is a [Groovy](https://groovy-lang.org) script where the network is exposed a global variable so that the user can write in Groovy the logic to generate contingency from network elements.

### Filter server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/filter-server
- Storage: PostGresSQL
- Connected to message broker:
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for storing and instanciating filters. Filters are just set of network elements and are useful in many places in the application: to help configuring network modifications, to filter data in the UIs, etc. The overall design of filters is very close to contingency lists: there is form based and Groovy script based filters.

### Security analysis server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/security-analysis-server
- Storage: Cassandra DB
- Connected to message broker: consumer and producer
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for running a security analysis and storing resulting limit violations. A security analysis is based on loadflow calculation on a set of post contingency state. When running a security analysis we need a network ID from the network store server, a contingencu list ID from the action server and then we get a result unique ID. Thanks to this result ID we can get the calculation status (running, complete, failed, etc) and then once calculation is complete we can get full limit violation results. As for loadflow calculation we rely on [PowSyBl security analysis API](https://github.com/powsybl/powsybl-core/tree/main/security-analysis/security-analysis-api) and we support two implementations, one based on [Hades2](https://rte-france.github.io/hades2/) and another one based on [OpenLoadFlow](https://github.com/powsybl/powsybl-open-loadflow), [DynaFlow](https://github.com/dynawo/dynaflow-launcher) one will come soon. Security analysis are generally long time running calculation (like a few minutes), so a special design has been implemented to be able to run asynchronous calculation and allows performance scaling by running many server replicas. When a new calculation is asked using the REST API, a message is posted in a message broker queue, then a woker service (could have many instance of it across multiple replica of the service in the cluster) take the message and run the calculation. Once calculation is complete results are written to the database and a message is posted to another queue of the broker so that other web services can wait and be notified for a security analysis completion.

### Directory server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/directory-server
- Storage: PostGresSQL
- Connected to message broker: producer
- Other services dependencies: 
- Use PowSyBl libraries: no

This service is responsible for mananing a file system like hierarchy of directory where other kind of data managed by other micro-services (studies, contingency lists, filters called directory elements) can be referenced using their unique IDs. When added to a directory an element is associated to a name, a owner and a description. Access right is managed by only directories and is in the current version only implemented by a "private" attribute. A private directory is accessible with all its contents (some elements, or some sub-directories) only to its owner. A non private directory is implicitely a public directory and access is granted to all users.

### Dynamic simulation server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/dynamic-simulation-server
- Storage: Cassandra DB
- Connected to message broker: producer and consumer
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for running dynamic simulations. Is is based on [PowSyBl dynamic simulation API](https://github.com/powsybl/powsybl-core/tree/main/dynamic-simulation/dynamic-simulation-api) and [Dynawo implementation](https://github.com/powsybl/powsybl-dynawo). This is still an on going work and it is not yet fully working with the current version of the code. This is also not integrated into GridStudy.

### Dynamic mapping server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/dynamic-mapping-server
- Storage: PostGresSQL
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for configuring how dynamic data is mapped to a network. This is still an ongoing work and is not yet integrated with dynamic simulation server.

### Study server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/study-server
- Storage: PostGresSQL
- Connected to message broker: producer and consumer
- Other services dependencies: network store server, case server, single line diagram server, network conversion server, geo data server, network map server, network modification server, loadflow server, actions server, security analysis server, report server
- Use PowSyBl libraries: yes

This services is responsible for managing studies and this is the main back-end entry point for GridStudy. This is why this services, can make requests to many other services. A study is created from the import of a case which is stored in the network store server. A study is then persisted to database with a link to the imported network. One of the greatest feature of study server is the ability to create new variants of the initial network and run simulations on it. So, we have a network modification tree embedded into each study.  We are able to create new nodes, either a modification node or a calculation node. A modification node is associated to a group of modification (stored in the modification server), it can be 'built' i.e. instantiated by taking previous network and applying the modifications. This network building heavily rely on network store variant feature. A calculation node is associated to a list of simulator and can also be built by running the simulation. Simulation results (loadflow, security analysis) are linked to calculation node, to be displayed later in the GridStudy UI. Unlike most of the other web services, study server has been implemented using Spring WebFlux so using reative programming (Spring reactor). This is because as an entry point for all study related requests, it is expected to have a higher request throughput than for most of the other web service. The data access layer is based on classic JPA which is not reactive, so the implementation is not fully reactive. 

### Explore server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/explore-server
- Storage: no
- Connected to message broker:
- Other services dependencies: directory server, study server, actions server, filter server
- Use PowSyBl libraries: no

This service is responsible for managing users data. This is the back-end entry point for GridExplore front-end. The main goal of this service is to aggregate and coordinate requests to directory server and the other data services (int current version: study server, action server and filter server, more to come). 

### Config server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/config-server
- Storage: PostGresSQL
- Connected to message broker: producer
- Other services dependencies:
- Use PowSyBl libraries: no

This service is responsible for storing parameters coming from GridSuite front-ends. A parameter is a key value (String, String) type data associated to a user and an application (just a String like 'study' or 'merge'). Thanks to this service front-ends can saved their states (like options in parameters dialog) and users are able to keep same UI configuration whatever browser or machine is used to connect to GridSuite.

###  Balances adjustment server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/balances-adjustment-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for running a [balances adjustment](https://github.com/powsybl/powsybl-balances-adjustment) (coming from PowSyBl) on a network stored by the network store server. This is still an ongoing work as many thinks are still missing like getting net position from [PEFV files](https://eepublicdownloads.entsoe.eu/clean-documents/EDI/Library/cim_based/schema/PEVF_IG_v1.1.pdf) or getting generation shift keys from a [CRAC file](https://eepublicdownloads.entsoe.eu/clean-documents/EDI/Library/crac/CRAC_implementation_guide_v2r3.pdf).

### Merge orchestrator server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/merge-orchestrator-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, cgmes boundary server, case server, load flow server, balances adjustment server, network conversion server 
- Use PowSyBl libraries: yes

This service is responsible for implementing an ENTSOE merging process. It could be based on UCTE files our CGMES files. Merge orchestrator could support many merging configuration at the same time. A merging configuration is a type of process (D-1, D-2, etc) and a list of TSO to merge. Once configuration is created, the merge orchestrator wait for required files (called IGM)  to be available (using the RabbitMQ case queue, feed by case server when a new case is available) and when all files are here, start a merging process. A merging process begins by importing IGM to network store (so a conversion to IIDM is done), then each IGM is validated using the case validation server, then a topological merge is done (creating a CGM) and finally either a loadflow or a balances adjustment is ran on the CGM. The CGM is then ready for download using the GridMerge interface. The goal is also to be able to publish the CGM ate the end of the process (not yet done). Merge orchestrator server also supports complex IGM replacement strategies (to handle missing cases) using a high configurable Groovy script. 

### Case validation server

- Kind: Web service with a REST API
  - Source repository: https://github.com/gridsuite/case-validation-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, loadflow server
- Use PowSyBl libraries: yes

This services is responsible for case validation according to Entsoe EMF specifications and is used by the merge orchestrator server.

### CGMES boundary server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/cgmes-boundary-server
- Storage: no
- Connected to message broker: no
- Other services dependencies:
- Use PowSyBl libraries: no

This service is responsible for storing CGMES boundary files, needed when converting a CGMES file to an IIDM network. Boundary files have 2 profiles EQ and TP and have unique ID. Each time a new version of a boundary file is published by ENTSOE, a new ID is used so that we can keep with this service the full history of boundaries.

### Study notification server

- Kind: WebSocket endpoint
- Source repository: https://github.com/gridsuite/notification-server
- Storage: no
- Connected to message broker: yes a consumer of study queue
- Other services dependencies:
- Use PowSyBl libraries: no

This service is responsible to filtering and forwarding messages from study queue of the Broker (RabbitMQ) to a WebSocket endpoint. This WebSocket enpoint is used by GridStudy front-end for asynchronous update of UI component. Only part of the back-end study, relevant for UI update is forwarded. 

### Directory notification server

- Kind: WebSocket endpoint
- Source repository: https://github.com/gridsuite/directory-notification-server
- Storage: no
- Connected to message broker: yes a consumer of directory queue
- Other services dependencies:
- Use PowSyBl libraries: no

Same as study notification server, but for GridExplore asynchronous update.

### Config notification server

- Kind: WebSocket endpoint
- Source repository: https://github.com/gridsuite/config-notification-server
- Storage: no
- Connected to message broker: yes a consumer of config queue
- Other services dependencies:
- Use PowSyBl libraries: no

Same as study notification server, but to manage config-server notifications. Is is used by all front-ends.

### Merge notification server

- Kind: WebSocket endpoint
- Source repository: https://github.com/gridsuite/merge-notification-server
- Storage: no
- Connected to message broker: yes a consumer of merge queue
- Other services dependencies:
- Use PowSyBl libraries: no

Same as study notification server, but for GridMerge asynchronous update.

### Gateway

- Kind: Gateway
- Source repository: https://github.com/gridsuite/gateway
- Storage: no
- Connected to message broker: no
- Other services dependencies: study server, action server, filter server, directory server, case server
- Use PowSyBl libraries: no

This is the only entry point to the back-end. So front-ends can only send requests to the gateway. Request are then routed to other micro services. This gateway is mainly responsible from implementing secury features: https and user access right verification.

### Case import job

- Kind: Cron job
- Source repository: https://github.com/gridsuite/case-import-job
- Storage: Cassandra DB
- Connected to message broker: yes as a message producer
- Other services dependencies:
- Use PowSyBl libraries: yes

This cron job is responsible for regularly querying a FTP server, to get new cases (in a specific configured directory of the ftp). If new cases are available, they are all uploaded into the case server, so we can consider, this job as a bridge between an external FTP and our internal case storage solution. This job keep track of already imported cases by storing meta infos in a database (which is updated each time new cases are uploaded). Only cases supported by PowSyBl framework are processed.

### CGMES boundary import job

- Kind: Cron job
- Source repository: https://github.com/gridsuite/cgmes-boundary-import-job
- Storage: no
- Connected to message broker: yes as a message producer
- Other services dependencies:
- Use PowSyBl libraries: no

This cron job is responsible for regularly querying a FTP server to get new CGMES boundary files and upload to to CGMES boundary server.

### CGMES assembling job

- Kind: Cron job
- Source repository: https://github.com/gridsuite/cgmes-assembling-job
- Storage: Cassandra DB
- Connected to message broker: yes as a message producer
- Other services dependencies:
- Use PowSyBl libraries: yes

This cron job and a variant of case import job. The main difference is that this job is able process CGMES cases, profile by profile. So it has a knowledge of CGMES profiles structure and keep track for each case of which profile is available or not yet. When all profiles are ready, it build a full consistent CGMES case (a zip file) and upload it to case server.

## Front-ends description:

###  Grid Explore

TODO

### Grid Study

TODO

### Grid Merge

TODO

### Grid Dyna

TODO

### Grid Geo

Not yet implemented
