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
- Depends on other Web services: none

This service is responsible for storing network in [IIDM](https://www.powsybl.org/pages/documentation/grid/model/) format. Each of the network elements is stored in its own table indexed by its ID and voltage level ID allowing 3 levels of queries for a given network: all elements, an element from its ID, all elements belonging to a substation voltage level. This web services is not intended to be accessed directly though its REST API but using the Java Client. The java Client is an alternative implementation of the [IIDM API](https://github.com/powsybl/powsybl-core/tree/main/iidm/iidm-api) (alternative regarding the original in memory one from PowSyBl) and can be found in the network-store-iidm-impl module of the repository. An integration of the Java client into Spring framework is also available int he network-store-client module.

### Case server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-case
- Storage: file system and ElasticSearch
- Connected to message broker: yes, message producer
- Depends on other Web services: no

This service is responsible for storing case in any of PowSyBl supported formats (CGMES, UCTE, XIIDM, etc). Cases are just files and are store in the current version on a file system. The goal is to migrate to an Object Storage solution (like minIO) and to access it via a standard API for instance Amazon S3 API. Cases can be stored in 2 differents spaces: one "public" and one "private". Cases from the public space are accessible to any other services and all public cases can be listed and loaded without any restriction. Cases from private space are assigned to a unique identifier (UUID) when loaded to the case server and can only be accessed knowing this unique identifier. There is no way to list private cases. When a new case is uploaded to the server, it is indexed into an ElasticSearch database so that we can query the case server (using the REST API) for searching a case matching some provided criteria. In the current version, we only support simple index attribute coming from the case file name (like date, TSO, country, etc). In a future version, we will add more index attributes coming from the case content (like network element IDs, consumption or generation level, etc). Also, a message is sent to the broker each time a case is uploaded, so that other services can trigger some processing as soon as a new case that meet some criteria enter into the system.

### Report server

- Kind: Web service with a REST API
- Source repository: https://github.com/gridsuite/report-server
- Storage: PostGresSQL
- Depends on other Web services: no

### Network conversion server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-network-conversion-server
- Storage: none
- Connected to message broker: no
- Depends on other Web services: network store server, case server and report server

This service is responsible for importing cases to and exporting cases from the network store. Supported case importers and exporters are [the ones available from PowSyBl](https://www.powsybl.org/pages/documentation/index.html#grid-formats). The case is provided to the API either by providing the full case file or by just refering to a case in the case server. During the case conversion, so called 'functionnal logs' are generated. Functional logging is handled by [PowSyBl reporter API](https://github.com/powsybl/powsybl-core/tree/main/commons/src/main/java/com/powsybl/commons/reporter). Once conversion has been done and network fully stored to network store server, functional logs are stored to the report server for later analyse, search or UI display.

### Single line diagram server

- Kind: Web service with a REST API
- Source repository: https://github.com/powsybl/powsybl-single-line-diagram-server
- Storage: none
- Connected to message broker: no
- Depends on other Web services: network store server

This service is responsible for generating single line diagrams either for a voltage level or for a full substation from a network stored into network store server. Single line diagram generation feature is provided by a [PowSyBl component](https://github.com/powsybl/powsybl-single-line-diagram). A single line diagram is a SVG file and additional metadata (mainly to keep links with network elements).

### Geo data server

### CGMES GL server

### ODRE server

### Study server

### Network map server

### Network modification server

### Gateway

### Case import job

### Load flow server

### Notification server

###  Balances adjustment server

### Merge orchestrator server

### Case validation server

### Merge notification server

### CGMES assembling job

### CGMES boundary server

### Action server

### Security analysis server

### Config server

### Config notification server

### Directory server

### Dynamic simulation server

### CGMES boundary import job

### Filter server

### Dynamic mapping server

### Directory notification server

### Explore server



## Front-ends description:

###  Grid Explore

### Grid Study

### Grid Merge

### Grid Dyna

### Grid Geo
