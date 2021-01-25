# Securing access to Consul

## Status

** proposed **

## Context

This ADR defines the motiviation and approach used to secure access
to the Consul component in the EdgeX archticture
for *security-enabled configurations only*.
(Non-secure configuations continue to use Consul in
anonymous read-write mode.)

Consul provides several services for the EdgeX architecture:

- Service registry (see ADR in references below)
- Service health monitoring
- Mutable configuration data

Use of the services provided by Consul is optional on a service-by-service basis.
Use of the registry is controlled by the `-r` or `--registry` flag provided to an EdgeX service.
Use of mutable configuration data is controlled by the `-cp` or `--configProvider` flag provided to an EdgeX service.
When Consul is enabled as a configuration provider,
the `configuration.toml` is parsed into individual settings
and seeded into the Consul key-value store on the first start of a service.
Configuration reads and writes are then done to Consul if it is specified as the configuration provider,
otherwise the static `configuration.toml` is used.
Writes to the `[Writable]` section in Consul trigger per-service callbacks
notifying the application of the changed data.

Since configuration data can affect the runtime behavior of services,
compensating controls must be introduced in order to mitigate the risks introduced
by moving configuration from a static file into to an HTTP-accessible service with mutable state.

The current practice is that Consul is exposed via unencrypted HTTP in anonymous read/write mode
to all processes and EdgeX services running on the host machine.

## Decision

Consul will be configured with access control list (ACL) functionality enabled,
and each EdgeX service will utilize a Consul access token to authenticate to Consul.
Consul access tokens will be requested from the Vault Consul secrets engine
(to avoid introducing additional bootstrapping secrets).

DNS will be disabled via configuration as it is not used in EdgeX.

**Consul Access Via API Gateway**

Consul can be configured to be exposed via the EdgeX API Gateway
in order to provide remote configuration capabilities.
The UI can also be rebased (in theory) via the `ui_content_path`
[setting](https://www.consul.io/docs/agent/options#ui_config_enabled)
in order to be compatible with the API Gateway pathing conventions.

Consul supports authentication via the `X-Consul-Token` HTTP header,
the `Authorization: Bearer` header,
or via a `?token=` query parameter (insecure).
The API gateway should be configured to simply pass HTTP requests
and rely on Consul's authentication mechanims for both API and UI accesses.
For API accesses, the `X-Consul-Token` method should be used
as it will not conflict with API Gateway authentication.
For UI accesses, the Consul UI will prompt the user for an access token.


## Consequences

Full implementation of this ADR will deny Consul access to all existing Consul clients.
To limit the impacts of the change, deployment will take place in phases:

### Phase 1

- Vault bootstrapper will install Vault Consul secrets engine.
- Secretstore-setup will create a Vault token for consul secrets engine configuration.
- Consul will be started with Consul ACLs enabled with persistent agent tokens and a default "allow" policy.
- Consul bootstrapper will create a bootstrap management token
  and use the provided Vault token to (re)configure the Consul secrets engine in Vault.
- Consul bootstrapper will create an agent ("node") policy and token and install it into the agent.
- Create a global management token (stored in a file) for use in remote configuration.
- (Docker-only) Open a port to signal that Consul bootstrapping is completed.
  (Integrate with `ready_to_run` signal.)

### Phase 2

- Consul bootstrapper will install a role in Vault that creates global-management tokens in Consul with no TTL.
- Registry and configuration client libraries will be modified to accept a Consul access token.
- go-mod-bootstrap will have contain the necessary glue logic to
  request a service-specifc Consul access token from Vault
  and persist it as a service-specific secret.
- Consul configuration will be changed to a default "deny" policy
  once all services have been changed to authenticated access mode.

### Phase 3

- Introduce per-service roles and ACL policies that give each service
  access to its own subset of the Consul key-value store
  and to register in the service registry.
- Vault configuration will be changed to request two (2) hour expiring Consul access tokens
  instead of non-expiring global management tokens.
- Consul access tokens will be scoped to the needs of the particular service
  (ability to update that service's registry data, an access that services's KV store).
- Glue logic will ensure that expired Consul tokens are replaced with fresh ones.


## References

- [ADR for secret creation and distribution](./0008-Secret-Creation-and-Distribution.md)
- [ADR for secure bootstrapping](./0009-Secure-Bootstrapping.md)
- [ADR for service registry](https://github.com/edgexfoundry/edgex-docs/pull/283)
- [Hashicorp Vault](https://www.vaultproject.io/)
