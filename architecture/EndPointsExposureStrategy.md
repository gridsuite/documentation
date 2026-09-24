# Endpoints Exposure Strategy

The backend of the application is built on a micro-services architecture. To maintain control over external access, only a selected set of endpoints are exposed to the outside world and front-end applications.

## Principles

- All external connections are routed through a gateway, which serves as the sole entry point to the application. The gateway handles request routing and authentication.
- Only specific micro-services expose their endpoints via the gateway. These micro-services are responsible for enforcing user authorization.
- User authorization rules are defined in the User Admin Server for group definitions and in the Directory Server for access rights on resources (Read, Write, Manage), either by user or by group*.
- The exposed endpoints are consumed by both front-end applications and external clients (no separate gateway exists for external applications).
- The endpoints of the form `/v<n>/supervision/**` are not accessible from outside; they are maintenance endpoints. The filtering is done in the gateway.

This strategy is not yet fully implemented, work is in progress.

*This is the current situation; it may be possible in the future to add new services for other business authorization needs.


## Micro-Services Exposed via the Gateway

- Case Import Server
- Config Notification Server
- Config Server
- Directory Notification Server
- Dynamic Mapping Server
- Explore Server
- Monitor Server
- Monitor Notification Server
- Study Notification Server
- Study Server
- User Admin Server

## Authorization system

The exposed micro-services enforce authorization at endpoint level using Spring Security.

* Each exposed endpoint explicitly defines its authorization requirements. Access-controlled endpoints rely on the `AuthorizationService`, which delegates permission checks to the directory-server. Endpoints that do not require authorization explicitly declare that no authorization check is needed.
* User identity and roles are propagated through the Spring Security context. The `userId` and `roles` provided by the gateway are extracted from incoming request headers and made available throughout the application.
* User identity and roles are automatically propagated to downstream micro-services through the headers of outgoing requests, ensuring that authorization information is preserved across service-to-service calls.
* The security configuration is automatically validated by tests to ensure that all endpoints have an explicit authorization declaration.
* When interacting directly with a micro-service through Swagger, outside of the gateway, the user identity and roles must be provided explicitly.