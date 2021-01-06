# Securing access to Consul

## Status

** proposed **

## Context

This ADR defines the motiviation and approach used to secure access
to the Consul component in the EdgeX archticture.

Consul provides several services for the EdgeX architecture:

- Service registry
- Service health monitoring
- Mutable configuration data

Use of the services provided by Consul is optional on a service-by-service basis.
Use of the registry is controlled by the `--registry` flag provided to an EdgeX service.
Use of mutable configuration data is controlled by the `-cp` flag provided to an EdgeX service.
Where possible, a static `configuration.toml` file bundled with each service is used as a fallback mechanism.
When Consul is enabled as a configuration provider,
the `configuration.toml` is parsed into individual settings
and seeded into the Consul key-value store on the first start of a service.
Further accesses to configuration data are performed first against the Consul key-value store,
and the static `configuration.toml` remains as the fallback mechanism.

Since configuration data can affect the runtime behavior of services,
compensating controls must be introduced in order to mitigate the risks introduced
by moving configuration from a static file verified with a SHA-256 hash
into to an HTTP-accessible service with mutable state.

The current practice is that Consul is exposed via HTTP in anonymous read/write mode
to all processes and EdgeX services running on the host machine.

## Decision

Consul will be configured with access control list (ACL) functionality enabled,
and each EdgeX service will utilize a Consul access token to authenticate to Consul.
Consul access tokens will be requested from the Vault Consul secrets engine
(to avoid introducing additional bootstrapping secrets).

## Consequences

Full implementation of this ADR will deny Consul access to all existing Consul clients.
To limit the impacts of the change, deployment will take place in phases:

### Phase 1

- Vault bootstrapper will install Vault Consul secrets engine.
- Secretstore-setup will create a Vault token for consul secrets engine configuration.
- Consul will be started with Consul ACLs enabled with persistent agent tokens and a default "allow" policy.
- Consul bootstrapper will create a bootstrap management token
  and use the provided Vault token to (re)configure the Consul secrets engine in Vault.
- Consul bootstrapper will create agent ("node") token and install it into the agent.
- Open a port to signal that Consul bootstrapping is completed.
  (Integrate with `ready_to_run` signal.)

### Phase 2

- Consul bootstrapper will install a role in Vault that creates global-management tokens in Consul with no TTL.
- Registry and configuration client libraries will be modified to request
  Consul access tokens from Vault and use the Consul access token for Consul transactions.
- Consul tokens once requested will be stored in Vault KV store for later retrieval.
- Consul configuration will be changed to a default "deny" policy.

### Phase 3

- Introduce per-service roles and ACL policies that give each service
  access to its own subset of the Consul key-value store
  and to register in the service registry.
- Token lifetime will be reduced to 1 hour.
- Expiration of a Consul token will result in a call to Vault to request a new token
  (resulting token will be persisted in Vault KV store for the service).

## References

- [ADR for secret creation and distribution](./0008-Secret-Creation-and-Distribution.md)
- [ADR for secure bootstrapping](./0009-Secure-Bootstrapping.md)
- [Hashicorp Vault](https://www.vaultproject.io/)
