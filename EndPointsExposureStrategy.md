# Endpoints Exposure Strategy

The backend of the application is built on a micro-services architecture. To maintain control over external access, only a selected set of endpoints are exposed to the outside world and front-end applications.

## Principles

- All external connections are routed through a gateway, which serves as the sole entry point to the application. The gateway handles request routing and authentication.
- Only specific micro-services expose their endpoints via the gateway. These micro-services are responsible for enforcing user authorization.
- User authorization rules are defined in the User Admin Server for group definitions and in the Directory Server for access rights on resources (Read, Write, Manage), either by user or by group*.
- The exposed endpoints are consumed by both front-end applications and external clients (no separate gateway exists for external applications).


This strategy is not yet fully implemented, work is in progress.

*This is the current situation; it may be possible in the future to add new services for other business authorization needs.


## Micro-Services Exposed via the Gateway

- User Admin Server
- Config Server
- Explore Server
- Study Server
- Monitor Server
- Dynamic Mapping Server
- Directory Notification Server
- Study Notification Server
- Config Notification Server
