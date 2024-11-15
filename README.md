# GridSuite documentation

## Architecture

Here is a micro-service architecture diagram. Yellow boxes are micro-services, orange boxes are user interfaces and all blue boxes are external services (like databases, a message broker, etc). Arrows represent http, websockets or AMPQ connections. We use 2 different databases: PostgreSQL which is the default solution and ElasticSearch for data indexing. In addition to databases, we also use file systems and FTP storages. Most of the micro-services communications rely on synchronous http request (REST APIs), but we also have asynchronous communication thought a RabbitMQ message broker. Back-end access is done from front-end using http call to REST APIs and WebSocket connections for asynchronous updates.



![gridsuite_architecture.drawio](diagrams/gridsuite_architecture.svg)

## Technical stack

### Back-end

All of the micro-services rely mainly on language and frameworks:

- Java 11
- Spring Boot 2.6 + Spring Cloud 2021
- PowSyBl dependencies 1.3.0

And technical components:

- RabbitMQ 3.11
- PostgreSQL 13
- ElasticSearch 7.9

The application can be deployed via:

- Kubernetes 1.19+. 
- Docker compose 3


### Front-end

Front-ends are web application based on:

- ReactJS 18
- React Redux
- Component library: MUI 5
- DeckGL (WebGL based) for large power grid visualisation 
- OpenID Connect for authentication



## Micro-services description

### Network store server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-network-store
- Storage: PostgreSQL
- Connected to Message broker: no
- Other services dependencies: none
- Use PowSyBl libraries: yes

This service is responsible for storing network in [IIDM](https://www.powsybl.org/pages/documentation/grid/model/) format. Each of the network elements is stored in its own table indexed by its ID and voltage level ID allowing 3 levels of query for a given network: all elements, an element from its ID, all elements belonging to a substation voltage level. This web service is not intended to be accessed directly though its REST API but rather using the Java Client. The java Client is an alternative implementation of the [IIDM Java API](https://github.com/powsybl/powsybl-core/tree/main/iidm/iidm-api) (alternative regarding the original in memory one from PowSyBl) and can be found in the [network-store-iidm-impl](https://github.com/powsybl/powsybl-core/tree/main/iidm/iidm-impl) module of the repository. An integration of the Java client into Spring framework is also available in the [network-store-client](https://github.com/powsybl/powsybl-network-store/tree/main/network-store-client) module.

### Case server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-case
- Storage: file system and ElasticSearch
- Connected to message broker: yes, message producer
- Other services dependencies: no
- Use PowSyBl libraries: yes

This service is responsible for storing cases in any of the PowSyBl supported formats (CGMES, UCTE, XIIDM, etc). Cases are just files and in the current version are stored on a file system. Our goal is to migrate to an object storage solution (like minIO) and to access it via a standard API like Amazon S3 API. Cases can be stored in 2 different spaces: one "public" and one "private". Cases from the public space are accessible to any other services and all public cases can be listed and loaded without any restriction. Cases from private space are assigned to a unique identifier (UUID) when loaded to the case server and can only be retrieved knowing this identifier. There is no way to list all private cases. When a new case is uploaded to the server, it is also indexed into an ElasticSearch database so that we can query the case server (using the REST API) for searching cases matching some provided criteria. In the current version, we only support simple index attributes coming from the case file name (like date, TSO, country, etc). In a future version, we will add more index attributes coming from the case content (like network element IDs, consumption or generation level, etc). Also, a message is sent to the RabbitMQ broker each time a case is uploaded, so that other services can trigger some processing as soon as a new case that meets some criteria enters into the system.

### Report server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/report-server
- Storage: PostgreSQL
- Other services dependencies: no
- Use PowSyBl libraries: yes

This service is responsible for storing functional logs. Functional logs unlike technical logs (managed by Slf4j/Logback and EFK stack) are designed to be presented to a user.  They are often less verbose than technical logs and contain meta-data associated to each log (for instance numerical variable values and their unit like MW or A). In the current version of the service, there are very few query capabilities but the aim is to be able to select logs from a time interval, from key types, etc. Each log is associated by a unique identifier (UUID) to a container which can be for instance a study node or a merge from the merge orchestrator server or any kind of application that need to store functional logs.  Functional  Java modeling logs are handled by [PowSyBl reporter API](https://github.com/powsybl/powsybl-core/tree/main/commons/src/main/java/com/powsybl/commons/reporter) and are used in all micro-services that need to create logs. Many of PowSyBl underlying features are capable of generating functional logs by just passing a  `Reporter` by parameter. On front-end side a [generic ReactJs component](https://www.npmjs.com/package/@gridsuite/commons-ui) is available to display functional logs and is already integrated in GridStudy and GridMerge.

### Network conversion server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-network-conversion-server
- Storage: none
- Connected to message broker: no
- Other services dependencies: network store server, case server and report server
- Use PowSyBl libraries: yes

This service is responsible for importing/exporting cases to/from the network store. Supported case importers and exporters are [the ones available from PowSyBl](https://www.powsybl.org/pages/documentation/index.html#grid-formats). The case is provided to the API either by providing the full case file (as binary data) or by just referencing to a case already uploaded to the case server. During the case conversion, functional logs are generated. Once conversion has been done and network fully stored to network store server, functional logs are sent to the report server for later analysis, search or visualization.

### Single line diagram server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-single-line-diagram-server
- Storage: none
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for generating single line diagrams either for a voltage level or for a full substation using a network stored into the network store server. Single line diagram generation feature is provided by a [PowSyBl component](https://github.com/powsybl/powsybl-single-line-diagram). A single line diagram consists of one SVG file and additional meta-data (mainly to keep links with network elements and original network IDs).

### Geo data server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/geo-data
- Storage: PostgreSQL
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for storing network related geographical data. There are two kinds of geographical data: substation data and line data. Substation geographical data are just one GPS (WGS84) coordinate associated to a substation ID. Line geographical data are a list of GPS coordinates corresponding to the position of all the pylons (so substation position at both side are not included) of the line. When data is queried for a specific network, missing data are automatically created. For substations, missing coordinates are computed from neighboring substations using a centroid algorithm. For lines, coordinates are not required by the front-end (a straight line is drawn in case of missing data) and different fixing algorithms are used in case of inconsistent data: for instance when the list of pylon coordinates does not allow the creation of a polyline (because of missing segments). 

### CGMES GL server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/cgmes-gl-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, geo data server, Case server
- Use PowSyBl libraries: yes

This service is responsible of converting CGMES GL (Graphical Layout) profile to GPS coordinates uploaded to geo data server. 

### ODRE server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/odre-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, geo data server
- Use PowSyBl libraries: no

This service is responsible for converting GPS coordinates of substations and lines coming from [Open Data Réseau Energies](https://opendata.reseaux-energies.fr/pages/accueil/?flg=fr) system (ODRE) and uploading them to the geo data server. ODRE is a web site (and also a web service with a REST API) where GPS positions of RTE's equipments have been open sourced. 

### Network map server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/network-map-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for extracting/filtering/reshaping network data from network store, so that it can be used on the front-end side to feed the network map UI, the spreadsheet and many other UI components that need to display network data (in GridStudy).

### Network modification server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/network-modification-server
- Storage: PostgreSQL
- Connected to message broker: no
- Other services dependencies: network store server, report server
- Use PowSyBl libraries: yes

This service is responsible for storing and applying modifications to a network of the network store server. This is one of the most important service of the application as it implements many business logic for modifying the network. There are many kinds of modifications supported by this service like network element creation (substation, voltage level, generator, load, etc), network element removal, switch opening or closing, line or transformer triggering or locking out, network element elementary modification (like a target value modification, or a tap position modification). Only a very small number of expected possible modifications have yet been implemented with many more to come in the future. Network modifications have a unique ID (UUID) and could be associated to a group which has itself a unique ID (UUID) and this is how a set of network modifications is referenced by other web services. Depending on the use case a network modification can be stored only or can be both stored and applied to a network  (given a network ID in the network store) or just selected from the database and applied to a network. When a modification is applied to a network functional logs are created to give more informations to the users and sent to the report server for later use.

### Load flow server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/loadflow-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, report server
- Use PowSyBl libraries: yes

This service is reponsible for running a [loadflow](https://en.wikipedia.org/wiki/Power-flow_study) on a network of the network store server. It relies on [PowSyBl loadflow API](https://github.com/powsybl/powsybl-core/tree/main/loadflow/loadflow-api), a generic implementation agnostic API. As a consequence, as soon as a new implementation is available, it can be integrated to this service without adding any new line of code. In the current version, two loadflow implementations are integrated: [Hades2](https://rte-france.github.io/hades2/) which is a free software made by RTE and [OpenLoadFlow](https://github.com/powsybl/powsybl-open-loadflow) a loadflow fully implemented in PowSyBl. A third one, [DynaFlow](https://github.com/dynawo/dynaflow-launcher) is going to be integrated soon. When a loadflow has run on a network, we get results like status, metrics, functional logs and above all, state variables (voltages, flows) updated in the network store.  

### Action server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/actions-server
- Storage: PostgreSQL
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for storing and instantiating contingency lists. Contingencies are used by security analysis to describe network elements that we want to simulate the loss of. We support two main types of contingency lists:  form-based and script-based. A form-based contingency list is described by an element type (generator, line, etc) and some filtering criteria (country, nominal voltage, etc), so it needs to be instantiated with a specific network to be used in a security analysis. A script-based contingency list is a [Groovy](https://groovy-lang.org) script where the network is exposed as a global variable so that the user can write in Groovy the logic to generate contingencies from network elements.

### Filter server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/filter-server
- Storage: PostgreSQL
- Connected to message broker:
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for storing and instantiating filters. Filters are just sets of network elements and will be used in many places in the application: to help configuring network modifications, to filter data in the UIs, etc. The overall design of filters is very close to contingency lists: there are form-based and Groovy-script-based filters.

### Security analysis server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/security-analysis-server
- Storage: PostgreSQL
- Connected to message broker: consumer and producer
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for running a security analysis and storing resulting limit violations. A security analysis is based on loadflow calculations on a set of post-contingency states. When running a security analysis we need a network ID from the network store server, a contingency list ID from the action server and then we get a unique result ID. Thanks to this result ID we can get the calculation status (running, complete, failed, etc) and then once the calculation is complete we can get full limit violation results. As for loadflow calculation we rely on [PowSyBl security analysis API](https://github.com/powsybl/powsybl-core/tree/main/security-analysis/security-analysis-api) and we support two implementations, one based on [Hades2](https://rte-france.github.io/hades2/) and the other one based on [OpenLoadFlow](https://github.com/powsybl/powsybl-open-loadflow), [DynaFlow](https://github.com/dynawo/dynaflow-launcher) implementation will come soon. Security analyses are generally long time running calculations (few minutes length), this is why a special design has been implemented to be able to run asynchronous calculations and to allow performance scaling by running many server replicas. When a new calculation is asked using the REST API, a message is posted on a RabbitMQ queue, then a worker service (could have many instances of it across multiple replicas of the service in the cluster) takes the message and runs the calculation. Once the calculation is complete, results are written back to the database and a message is posted to another queue of the broker so that other web services can be notified when security analysis results are available.

### Sensitivity analysis server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/sensitivity-analysis-server
- Storage: PostgreSQL
- Connected to message broker: consumer and producer
- Other services dependencies: network store server, actions server, filter server, report server
- Use PowSyBl libraries: yes

This service is responsible for running a sensitivity analysis and storing resulting sensitivity factors, sensitivity values and contingency statuses. 
A sensitivity analysis is based on loadflow calculations on a list of sensitivity factors, a list of contingencies and a list of sensitivity variable sets. 
When running a sensitivity analysis we need a network ID from the network store server, and a configuration containing multiple filter ID 
from the filter server, multiple contingency list ID from the actions server and then we get a unique result ID. 
Thanks to this result ID we can get the calculation status (running, complete, failed, etc) and then once the calculation is complete we can get full results. 
As for loadflow calculation we rely on [PowSyBl sensitivity analysis API](https://github.com/powsybl/powsybl-core/tree/main/sensitivity-analysis-api)
and we support two implementations, one based on [Hades2](https://rte-france.github.io/hades2/) and the other one based 
on [OpenLoadFlow](https://github.com/powsybl/powsybl-open-loadflow). Sensitivity analyses are generally long time running calculations (few minutes length), this is why a special design has been implemented 
to be able to run asynchronous calculations and to allow performance scaling by running many server replicas. When a new calculation 
is asked using the REST API, a message is posted on a RabbitMQ queue, then a worker service (could have many instances of it across multiple replicas of the service in the cluster) 
takes the message and runs the calculation. Once the calculation is complete, results are written back to the database and a message is posted 
to another queue of the broker so that other web services can be notified when sensitivity analysis results are available.

### Short-circuit server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/shortcircuit-server
- Storage: PostgreSQL
- Connected to message broker: consumer and producer
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for running a short-circuit analysis and storing resulting short-circuit violations. As for loadflow calculation we rely on [PowSyBl shortcircuit API](https://github.com/powsybl/powsybl-core/tree/main/shortcircuit-api). A short-circuit computation model is required to be executed behind the API. As security and sensibility analyses, short-circuit calculations are ran asynchronously. When running a short-circuit analysis we need a network ID from the network store server and then we get a unique result ID. Thanks to this result ID we can get the calculation status (running, complete, failed, etc) and then once the calculation is complete we can get full violation results. When a new calculation is asked using the REST API, a message is posted on a RabbitMQ queue, then a worker service (could have many instances of it across multiple replicas of the service in the cluster) takes the message and runs the calculation. Once the calculation is complete, results are written back to the database and a message is posted to another queue of the broker so that other web services can be notified when short-circuit analysis results are available.

### Directory server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/directory-server
- Storage: PostgreSQL
- Connected to message broker: producer
- Other services dependencies: 
- Use PowSyBl libraries: no

This service is responsible for managing a file system like the hierarchy of directories where other kind of data managed by other micro-services (studies, contingency lists, filters called directory elements) can be referenced using their unique IDs. When added to a directory an element is associated to a name, an owner and a description. Access rights are only managed through directories and in the current version they are only implemented by a "private" attribute. A private directory is accessible with all its contents (some elements, or some sub-directories) to its owner only. A non private directory is implicitly a public directory and access is granted to all users.

### Time-series server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/timeseries-server
- Storage: PostgreSQL
- Connected to message broker: no
- Other services dependencies:
- Use PowSyBl libraries: yes

This service is responsible for storing time-series data. It is used to persist the output data of dynamic simulations which are displayed later in GridStudy.

### Dynamic simulation server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/dynamic-simulation-server
- Storage: PostgreSQL
- Connected to message broker: producer and consumer
- Other services dependencies: network store server, report server, dynamic mapping server, time-series-server
- Use PowSyBl libraries: yes

This service is responsible for running dynamic simulations. It is based on [PowSyBl dynamic simulation API](https://github.com/powsybl/powsybl-core/tree/main/dynamic-simulation/dynamic-simulation-api) and [Dynawo implementation](https://github.com/powsybl/powsybl-dynawo). 

### Dynamic security analysis server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/dynamic-security-analysis-server
- Storage: PostgreSQL
- Connected to message broker: producer and consumer
- Other services dependencies: network store server, report server, dynamic simulation server
- Use PowSyBl libraries: yes

This service is responsible for running dynamic security analysis. It is based on [PowSyBl dynamic security analysis API](https://github.com/powsybl/powsybl-core/tree/main/dynamic-security-analysis) and [Dynawo implementation](https://github.com/powsybl/powsybl-dynawo).

### Dynamic mapping server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/dynamic-mapping-server
- Storage: PostgreSQL
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for configuring how dynamic data is mapped to a network.

### Study server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/study-server
- Storage: PostgreSQL
- Connected to message broker: producer and consumer
- Other services dependencies: network store server, case server, single line diagram server, network conversion server, geo data server, network map server, network modification server, loadflow server, actions server, security analysis server, report server
- Use PowSyBl libraries: yes

This service is responsible for managing studies and it is the main back-end entry point for GridStudy. This is why this service can connect to many other services. A study is created from the import of a case which is then stored in the network store server. A study is then persisted to database with a link to the imported network. One of the greatest feature of study server is the ability to create new variants of the initial network and run simulations on it. So, we have a network modification tree model associated to each study.  We are able to create new nodes, either a modification node or a calculation node. A modification node is associated to a group of modifications (stored in the modification server), it can be 'built' i.e. instantiated by taking previous network and applying the modifications. This network build relies on IIDM variant feature. A calculation node is associated to a list of simulator and can also be built by running the simulations. Simulations results (loadflow, security analysis) are linked to calculation node, to be displayed later in the GridStudy UI. Unlike most of the other web services, the study server has been implemented using Spring WebFlux so using reactive programming (Spring reactor). As an entry point for all study related requests, it is expected to have a higher request throughput than most of the other web services. The data access layer is based on classic JPA which is not reactive, so the implementation is not fully reactive. 

### Explore server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/explore-server
- Storage: no
- Connected to message broker:
- Other services dependencies: directory server, study server, actions server, filter server
- Use PowSyBl libraries: no

This service is responsible for managing users data. This is the back-end entry point for GridExplore front-end. The main goal of this service is to aggregate and  coordinate requests to the directory server and the other data services (in the current version: study server, action server and filter server, more to come). 

### Config server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/config-server
- Storage: PostgreSQL
- Connected to message broker: producer
- Other services dependencies:
- Use PowSyBl libraries: no

This service is responsible for storing parameters coming from GridSuite front-ends. A parameter is a key value (String, String) type data associated to a user and an application (just a String like 'study' or 'merge'). Thanks to this service, front-ends can have their states saved (like options in parameters dialog) and users are able to keep the same UI configuration on different machines and browsers.

### Spreadsheet config server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/spreadsheet-config-server
- Storage: PostgreSQL
- Connected to message broker: no
- Other services dependencies: none
- Use PowSyBl libraries: no

This service is responsible for storing and managing spreadsheet configurations. A spreadsheet configuration is a set of columns that can be used to display network data in a tabular form. This service is used by GridStudy and GridExplore front-ends.
###  Balances adjustment server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/balances-adjustment-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server
- Use PowSyBl libraries: yes

This service is responsible for running a [balances adjustment](https://github.com/powsybl/powsybl-balances-adjustment) (coming from PowSyBl) on a network stored by the network store server. This is still an ongoing work as many functionalities are still missing like getting net positions from a [PEFV files](https://eepublicdownloads.entsoe.eu/clean-documents/EDI/Library/cim_based/schema/PEVF_IG_v1.1.pdf) or getting generation shift keys from a [CRAC file](https://eepublicdownloads.entsoe.eu/clean-documents/EDI/Library/crac/CRAC_implementation_guide_v2r3.pdf).

### Merge orchestrator server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/merge-orchestrator-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, cgmes boundary server, case server, load flow server, balances adjustment server, network conversion server 
- Use PowSyBl libraries: yes

This service is responsible for implementing an ENTSOE merging process. It could be based on UCTE files or CGMES files. Merge orchestrator could support several merging configurations at the same time. A merging configuration is a type of process (D-1, D-2, etc) and a list of TSOs to merge. Once the configuration is created, the merge orchestrator waits for required files (called IGM)  to be available (using the RabbitMQ case queue and fed by case server when a new case is available) and when all files are here, starts a merging process. A merging process begins by importing IGMs to network store (meaning that a conversion to IIDM is done), then each IGM is validated using the case validation server, then a topological merge is done (creating a CGM) and finally either a loadflow or a balances adjustment is run on the CGM. The CGM is then ready for download using the GridMerge interface. The goal is also to be able to publish the CGM at the end of the process (to be implemented). The Merge orchestrator server also supports complex IGM replacement strategies (to handle missing cases) using a highly configurable Groovy script. 

### Case validation server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/case-validation-server
- Storage: no
- Connected to message broker: no
- Other services dependencies: network store server, loadflow server
- Use PowSyBl libraries: yes

This service is responsible for case validation according to Entsoe EMF specifications and is used by the merge orchestrator server.

### CGMES boundary server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/cgmes-boundary-server
- Storage: no
- Connected to message broker: no
- Other services dependencies:
- Use PowSyBl libraries: no

This service is responsible for storing CGMES boundary set files, needed when converting a CGMES file to an IIDM network. Boundary set files have 2 profiles EQ and TP and have a unique ID. Each time a new version of a boundary file is published by the ENTSOE, a new ID is used so that we can keep with this service the full history of boundaries.

### Study notification server

- Kind: WebSocket endpoint
- Source repository: https://github.com/gridsuite/notification-server
- Storage: no
- Connected to message broker: yes a consumer of study queue
- Other services dependencies:
- Use PowSyBl libraries: no

This service is responsible for filtering and forwarding messages from 'study' queue of the RabbitMQ broker to a WebSocket endpoint. This WebSocket endpoint is used by the GridStudy front-end for asynchronous update of the UI component. Only part of the back-end study, relevant for UI update is forwarded. 

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

Same as study notification server, but to manage config-server notifications. It is used by all front-ends.

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
- Other services dependencies: nearly all other web services
- Use PowSyBl libraries: no

This is the only entry point to the back-end. Front-ends can only send requests to the gateway. All the other web services do not expose their API directly to the front-ends. Requests are routed by the gateway to other micro services. This gateway is mainly responsible for implementing security features: https and user access rights verification.

### User admin server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/user-admin-server
- Storage: PostgreSQL
- Connected to message broker: no
- Other services dependencies: no
- Use PowSyBl libraries: no

This service is responsible for storing a list of users authorized to access the application and storing also users connection information.
It is used by the gateway service to check access authorization.

### User identity OIDC replication server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/user-identity-oidc-replication-server
- Storage: PostgreSQL
- Connected to message broker: no
- Other services dependencies: no
- Use PowSyBl libraries: no

This service is responsible for storing a list of users information.
The gateway use this API to store users information coming from oidc profiles in idtokens.
It is then possible to retrieve users information given a sub

### Case import job

- Kind: Cron job
- Source repository: https://github.com/gridsuite/case-import-job
- Storage: PostgreSQL
- Connected to message broker: yes as a message producer
- Other services dependencies:
- Use PowSyBl libraries: yes

This cron job is responsible for regularly querying a FTP server, to get new cases (in a specific configured directory of the ftp). When new cases are available, they are all uploaded into the case server, so we can consider this job as a bridge between an external FTP and our internal case storage solution. This job keeps track of already imported cases by storing meta infos in a database (which is updated each time new cases are uploaded). Only cases supported by PowSyBl framework are processed.

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
- Storage: PostgreSQL
- Connected to message broker: yes as a message producer
- Other services dependencies:
- Use PowSyBl libraries: yes

This cron job is a variant of the case import job. The main difference is that this job is able to process CGMES cases, profile by profile. So it has the knowledge of CGMES profiles structure and keeps track for each case on which profile is available or not yet. When all profiles are ready, it builds a full consistent CGMES case (a zip file) and uploads it to case server.

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
