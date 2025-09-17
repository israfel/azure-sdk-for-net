# AAD Hostname Construction for Data Plane SDKs

This document summarizes how Microsoft Entra ID (Azure AD) authentication determines the hostname (audience) for token requests in each Azure data-plane SDK listed in `dpa/scope.md`. The findings below focus exclusively on the Azure public cloud.

## Azure Storage (Blobs, Queues, Files Shares)

* **Audience selection:** Each client exposes an audience type (`BlobAudience`, `QueueAudience`, `ShareAudience`) with a default value of `https://storage.azure.com/`. This default is used to request tokens valid for any storage account. `Create*ServiceAccountAudience` helpers build an account-specific host such as `https://{account}.blob.core.windows.net/` by concatenating the storage account name with the service subdomain and Azure domain suffix.【F:sdk/storage/Azure.Storage.Blobs/src/Models/BlobAudience.cs†L33-L84】【F:sdk/storage/Azure.Storage.Queues/src/Models/QueueAudience.cs†L33-L84】【F:sdk/storage/Azure.Storage.Files.Shares/src/Models/ShareAudience.cs†L33-L84】
* **Scope construction:** When no explicit audience is provided, the clients call `AsPolicy` with the default storage scope (`https://storage.azure.com/.default`). If an audience is specified, `CreateDefaultScope` ensures the `/.default` suffix is appended to the host used for token acquisition.【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageClientOptions.cs†L63-L99】【F:sdk/storage/Azure.Storage.Common/src/Shared/Constants.cs†L132-L132】【F:sdk/storage/Azure.Storage.Blobs/src/BlobServiceClient.cs†L250-L259】
* **Challenge responses:** For bearer challenges the storage policy reads the `authorization_uri` to capture the tenant (segment after the first `/`) and appends `/.default` to the `resource_id` returned by the service to rebuild a full scope. This means the host originates from the challenged resource, e.g., a specific account endpoint.【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageBearerTokenChallengeAuthorizationPolicy.cs†L103-L125】

**Hostname meaning:**

* Scheme `https://` – Azure AD resource identifiers are always HTTPS.
* Host `storage.azure.com` – global storage resource when no account is targeted.
* Host `{account}.<service>.core.windows.net` – account name + storage service subdomain (`blob`, `queue`, `file`) + Azure public cloud suffix.

## Azure Data Tables

* **Audience selection:** `TableAudience` offers constants for several sovereign clouds; for the public cloud it chooses `https://storage.azure.com` (or `https://cosmos.azure.com` for premium endpoints). Custom values are allowed and used verbatim.【F:sdk/tables/Azure.Data.Tables/src/TableAudience.cs†L13-L86】
* **Scope construction:** `TableServiceClient` builds the policy with `audienceScope = (Audience ?? TableAudience.AzurePublicCloud).GetDefaultScope(...)`, which appends `/.default` unless the value already includes it. Premium (Cosmos) accounts flip the base host to the Cosmos domain, so the hostname embeds whether the tables live in Azure Storage or Cosmos DB.【F:sdk/tables/Azure.Data.Tables/src/TableServiceClient.cs†L264-L291】【F:sdk/tables/Azure.Data.Tables/src/TableAudience.cs†L66-L86】【F:sdk/tables/Azure.Data.Tables/src/TableConstants.cs†L8-L12】
* **Challenge responses:** When the service issues a challenge, the tables policy reads `authorization_uri` to discover the tenant ID (second URI segment), keeping the previously configured scopes for the host determined above.【F:sdk/tables/Azure.Data.Tables/src/TableBearerTokenChallengeAuthorizationPolicy.cs†L65-L88】

**Hostname meaning:**

* Scheme `https://` – Azure AD resource URL requirement.
* Host `storage.azure.com` – standard accounts backed by Azure Storage in the public cloud.
* Host `cosmos.azure.com` – Azure Cosmos DB-backed tables when the endpoint is premium.

## Azure Data App Configuration

* **Audience selection:** `ConfigurationClientOptions.GetDefaultScope` inspects the endpoint host suffix. For public cloud instances it selects `https://appconfig.azure.com`; custom audiences override this default.【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/ConfigurationClientOptions.cs†L73-L89】【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/AppConfigurationAudience.cs†L13-L56】
* **Client usage:** `ConfigurationClient` passes the computed scope into `BearerTokenAuthenticationPolicy`, so in the public cloud the hostname always resolves to the `appconfig.azure.com` resource.【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/ConfigurationClient.cs†L129-L148】

**Hostname meaning:**

* Scheme `https://` – Azure AD resource URL.
* Host `appconfig.azure.com` – App Configuration resource domain in the public cloud.

## Azure Key Vault (Secrets, Keys, Certificates, Administration)

* **Challenge-driven audience:** Clients install `ChallengeBasedAuthenticationPolicy`, which waits for a 401 challenge and reads the `resource` or `scope` parameter (e.g., `https://vault.azure.net`) to establish the hostname. The policy verifies that the requested host ends with the same domain and caches the result for the vault authority.【F:sdk/keyvault/Azure.Security.KeyVault.Shared/src/ChallengeBasedAuthenticationPolicy.cs†L118-L188】
* **Tenant discovery:** The policy also parses the `authorization` URI to extract the tenant ID from the path segments, tying it to the challenge cache key along with the hostname.【F:sdk/keyvault/Azure.Security.KeyVault.Shared/src/ChallengeBasedAuthenticationPolicy.cs†L163-L175】【F:sdk/keyvault/Azure.Security.KeyVault.Shared/src/ChallengeBasedAuthenticationPolicy.cs†L266-L288】

**Hostname meaning:**

* Scheme `https://` – enforced by the policy for security.
* Host `vault.azure.net` (or regional equivalents) – indicates the Key Vault/Managed HSM domain. Subdomains (`{vault-name}.vault.azure.net`) represent the specific vault.

## Azure Messaging Service Bus

* **Default scope:** Token-based clients use the constant `https://servicebus.azure.net/.default` as the scope when requesting AAD tokens, so the hostname is the global Service Bus resource identifier.【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Constants.cs†L40-L52】【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Authorization/ServiceBusTokenCredential.cs†L16-L88】
* **Namespace-specific URIs:** When building SAS audiences, `BuildAudienceResource` uses a `UriBuilder` around the fully qualified namespace to normalize `https://{namespace}.servicebus.windows.net` (lower-cased, path trimmed). This shows how the host components represent the customer namespace plus the service suffix.【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Administration/ServiceBusAdministrationClient.cs†L2703-L2738】
* **HTTP management requests:** The administration client requests tokens with `Constants.DefaultScope` whenever the credential is not SAS-based, reinforcing the same hostname for AAD.【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Administration/HttpRequestAndResponse.cs†L127-L136】

**Hostname meaning:**

* Scheme `https://` – Azure AD requirement.
* Host `servicebus.azure.net` – global AAD resource for Service Bus RBAC.
* Host `{namespace}.servicebus.windows.net` – specific namespace used when signing SAS tokens (not AAD).

## Azure Messaging Event Hubs

* **Default scope:** Event Hubs mirrors Service Bus, requesting tokens for `https://eventhubs.azure.net/.default` via `EventHubTokenCredential` and reusing that scope throughout AMQP operations.【F:sdk/eventhub/Azure.Messaging.EventHubs.Shared/src/Authorization/EventHubTokenCredential.cs†L16-L85】【F:sdk/eventhub/Azure.Messaging.EventHubs/src/Amqp/CbsTokenProvider.cs†L120-L168】

**Hostname meaning:**

* Scheme `https://`.
* Host `eventhubs.azure.net` – global Event Hubs resource identifier for RBAC.

## Azure Messaging Event Grid

* **Client scope:** `EventGridPublisherClient` configures a `BearerTokenAuthenticationPolicy` with the fixed scope `https://eventgrid.azure.net/.default`, so the hostname directly maps to the Event Grid service domain regardless of the topic’s regional subdomain.【F:sdk/eventgrid/Azure.Messaging.EventGrid/src/EventGridPublisherClient.cs†L53-L67】

**Hostname meaning:**

* Scheme `https://`.
* Host `eventgrid.azure.net` – global Event Grid RBAC resource.

## Summary Table

| SDK | How the hostname is derived | Hostname components |
| --- | --- | --- |
| Azure Storage (Blobs/Queues/Files) | Defaults to `https://storage.azure.com`, or `https://{account}.{service}.core.windows.net` when using account-specific audiences; challenges may return a `resource_id` that becomes the host before appending `/.default`.【F:sdk/storage/Azure.Storage.Blobs/src/Models/BlobAudience.cs†L33-L84】【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageBearerTokenChallengeAuthorizationPolicy.cs†L103-L125】 | `https` scheme; optional account name + service subdomain + Azure domain; `/.default` scope suffix. |
| Azure Data Tables | Chooses `https://storage.azure.com` for standard accounts or `https://cosmos.azure.com` for premium endpoints in the public cloud; `GetDefaultScope` appends `/.default`.【F:sdk/tables/Azure.Data.Tables/src/TableAudience.cs†L66-L86】【F:sdk/tables/Azure.Data.Tables/src/TableServiceClient.cs†L264-L291】 | `https` + `storage` (standard) or `cosmos` (premium) domain; `/.default` suffix. |
| Azure Data App Configuration | Inspects the endpoint host suffix to select the public-cloud resource `appconfig.azure.com` and appends `/.default`, or uses a custom audience.【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/ConfigurationClientOptions.cs†L73-L89】 | `https` + `appconfig.azure.com`; `/.default` scope suffix. |
| Azure Key Vault (Secrets/Keys/Certificates/Admin) | Waits for a challenge and adopts the `resource`/`scope` hostname (e.g., `https://vault.azure.net`), verifying it matches the requested domain; tenant derived from `authorization` URI.【F:sdk/keyvault/Azure.Security.KeyVault.Shared/src/ChallengeBasedAuthenticationPolicy.cs†L118-L188】 | `https` + vault domain; subdomain identifies specific vault instance. |
| Azure Messaging Service Bus | Uses constant `https://servicebus.azure.net/.default` for AAD tokens; SAS hosts are normalized via `UriBuilder` using the namespace.【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Constants.cs†L40-L52】【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Administration/ServiceBusAdministrationClient.cs†L2703-L2738】 | `https` + `servicebus.azure.net` for RBAC tokens; namespace + `servicebus.windows.net` for SAS. |
| Azure Messaging Event Hubs | Tokens requested for `https://eventhubs.azure.net/.default`.【F:sdk/eventhub/Azure.Messaging.EventHubs.Shared/src/Authorization/EventHubTokenCredential.cs†L16-L85】 | `https` + `eventhubs.azure.net`. |
| Azure Messaging Event Grid | Uses `https://eventgrid.azure.net/.default` regardless of topic endpoint host.【F:sdk/eventgrid/Azure.Messaging.EventGrid/src/EventGridPublisherClient.cs†L53-L67】 | `https` + `eventgrid.azure.net`. |

