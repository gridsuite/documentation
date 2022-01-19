# GridSuite documentation

## Architecture

Here is a micro-service architecture diagram. Yellow boxes are micro-services, orange boxes are user interfaces and everything is blue are external services (like databases, message broker, etc). Arrows represent http, websockets or AMPQ connections. We use 3 different databases: PostGresSQL which is the default solution, Cassandra DB for micro-services requiring high throughput and good scaling and ElasticSearch for data indexing. In addition to databases, we also use file system and FTP storages. Most of the micro-services communication rely on synchronous http request (REST APIs), but we also have asynchronous communication thought a RabbitMQ message broker. 



![gridsuite_architecture.drawio](diagrams/gridsuite_architecture.svg)

## Micro-services description

### Network store server

### Network conversion server

### Single line diagram server

### Case server

### CGMES GL server

### Geo data server

### Study server

### Network map server

### ODRE server

### Network modification server

### Gateway

### Grid Geo

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

### Report server

### Directory notification server

### Explore server

###  Grid Explore

### Grid Study

### Grid Merge

### Grid Dyna
