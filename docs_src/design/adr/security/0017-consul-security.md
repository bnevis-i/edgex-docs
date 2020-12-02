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
ompensating controls must be introduced in order to mitigate the risks introduced
by moving configuration from a static file verified with a SHA-256 hash
into to an HTTP-accessible service with mutable state.

The current practice is that Consul is exposed via HTTP in anonymous read/write mode
to all processes and EdgeX services running on the host machine.

## Decision

Consul will be configured with access control list (ACL) functionality enabled,
and each EdgeX service will utilize a Consul access token to authenticate to Consul.
Consul access tokens will be requested from the Vault Consul secrets engine
(to avoid introducing additional bootstrapping secrets).
Consul tokens will be transient and not persisted
(to avoid having to persistent store and renew generated tokens).

## Consequences

Full implementation of this ADR will deny Consul access to all existing Consul clients.
To limit the impacts of the change, deployment will take place in phases:

Phase 1:

- Vault bootstrapper will install Vault Consul secrets engine (one-time setup).
- Secretstore-setup will create a Vault token for consul secrets engine configuration.
- Consul bootstrapper will enable Consul ACLs with non-persistent tokens and a default "allow" policy.
- Consul bootstrapper will create a Consul management token,
  and use the provided Vault token to (re)configure the Consul secrets engine in Vault.
- Open a port to signal that Consul bootstrapping is completed.
  (Integrate with `ready_to_run` signal.)

Phase 2:

- Consul bootstrapping process will enable Consul ACLs with non-persistent tokens and change to a default "deny" policy.
- Consul bootstrapper will install a role in Vault that allows for all-Consul access with a 1 hour TTL.
- Registry and configuration client libraries will be modified to request
  Consul access tokens from Vault and use the Consul access token for Consul transactions.
  (Use of an expired token results in a new request to Vault.)

Phase 3:

- Introduce per-service roles and ACL policies that give each service
  access to its own subset of the Consul key-value store.


## References

- [ADR for secret creation and distribution](./0008-Secret-Creation-and-Distribution.md)
- [ADR for secure bootstrapping](./0009-Secure-Bootstrapping.md)
- [Hashicorp Vault](https://www.vaultproject.io/)
