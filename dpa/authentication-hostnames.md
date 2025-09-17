# API Endpoint and AAD Scope Hosts for Data Plane SDKs

This document reviews how each Azure data-plane SDK listed in `dpa/scope.md` determines (1) the API endpoint host that client requests target and (2) the Microsoft Entra ID scope host used when requesting access tokens in the Azure public cloud.

## Azure Storage (Blobs, Queues, File Shares)

**API endpoint host.** Parsing a storage connection string composes service endpoints such as `https://{account}.blob.core.windows.net` by combining the account name with service-specific hostname prefixes (`blob`, `queue`, `file`) and the default suffix `core.windows.net`, with the option to substitute a user-specified suffix or shared access signature when present.【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageConnectionString.cs†L830-L905】【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageConnectionString.cs†L995-L1027】【F:sdk/storage/Azure.Storage.Common/src/Shared/Constants.cs†L160-L178】

**AAD scope host.** Token credentials default to the shared `https://storage.azure.com/.default` scope, while `BlobAudience`, `QueueAudience`, and `ShareAudience` build resource-specific hosts (for example, `https://{account}.blob.core.windows.net/`) before appending `/.default`; challenge responses can override the host by returning a `resource_id` that the policy converts into the requested scope.【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageClientOptions.cs†L21-L99】【F:sdk/storage/Azure.Storage.Blobs/src/Models/BlobAudience.cs†L33-L84】【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageBearerTokenChallengeAuthorizationPolicy.cs†L103-L125】

## Azure Data Tables

**API endpoint host.** Table connection strings reuse the storage URI builder to emit primary and secondary hosts such as `https://{account}.table.core.windows.net`, swapping in custom suffixes or SAS tokens as needed. Premium (Cosmos DB-backed) accounts are detected by checking for `table.cosmos.azure.com` (or its legacy form), allowing the client to preserve that host when forming requests.【F:sdk/tables/Azure.Data.Tables/src/TableConnectionString.cs†L480-L528】【F:sdk/tables/Azure.Data.Tables/src/TableServiceClient.cs†L250-L291】【F:sdk/tables/Azure.Data.Tables/src/TableServiceClient.cs†L979-L984】

**AAD scope host.** The service selects an audience from `TableAudience` (defaulting to the storage resource) and calls `GetDefaultScope`, which appends `/.default` and flips to Cosmos-specific hosts when premium endpoints are detected. During a bearer challenge, the policy keeps the configured scopes while updating the tenant from the `authorization_uri` parameter.【F:sdk/tables/Azure.Data.Tables/src/TableServiceClient.cs†L272-L281】【F:sdk/tables/Azure.Data.Tables/src/TableAudience.cs†L66-L86】【F:sdk/tables/Azure.Data.Tables/src/TableBearerTokenChallengeAuthorizationPolicy.cs†L60-L83】

## Azure Data App Configuration

**API endpoint host.** Client construction stores the App Configuration endpoint parsed from the connection string (or provided directly) and reuses it for every request, so the API host mirrors the store’s DNS name.【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/ConfigurationClient.cs†L114-L148】【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/ConfigurationClient_private.cs†L37-L63】

**AAD scope host.** `ConfigurationClientOptions.GetDefaultScope` inspects the endpoint host suffix to choose the correct audience (`appconfig.azure.com` for the public cloud) before adding `/.default`, while `AppConfigurationAudience` supplies the canonical host values for each cloud.【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/ConfigurationClientOptions.cs†L73-L89】【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/AppConfigurationAudience.cs†L10-L57】

## Azure Key Vault (Secrets, Keys, Certificates, Administration)

**API endpoint host.** Key Vault clients capture the `vaultUri` supplied at construction and feed it into `KeyVaultPipeline`, which issues requests against that authority and exposes it via `VaultUri`. This host typically takes the form `{vault-name}.vault.azure.net` in the public cloud.【F:sdk/keyvault/Azure.Security.KeyVault.Keys/src/KeyClient.cs†L40-L82】【F:sdk/keyvault/Azure.Security.KeyVault.Shared/src/KeyVaultPipeline.cs†L17-L59】

**AAD scope host.** `ChallengeBasedAuthenticationPolicy` waits for a 401 challenge, extracts the `resource` or `scope` value (e.g., `https://vault.azure.net`), appends `/.default` when necessary, and validates that the host matches the requested vault domain before caching the scope with the tenant identifier.【F:sdk/keyvault/Azure.Security.KeyVault.Shared/src/ChallengeBasedAuthenticationPolicy.cs†L111-L178】

## Azure Messaging Service Bus

**API endpoint host.** `ServiceBusConnection` derives the fully qualified namespace from the connection string’s endpoint host (for example, `contoso.servicebus.windows.net`) and exposes the HTTPS service endpoint built from that namespace; SAS helpers normalize the namespace with an `https` scheme when constructing resource URIs.【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Primitives/ServiceBusConnection.cs†L23-L123】【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Administration/ServiceBusAdministrationClient.cs†L2703-L2734】

**AAD scope host.** All token credentials rely on the constant `https://servicebus.azure.net/.default` scope, and `ServiceBusTokenCredential` exposes helpers that request tokens using that default audience.【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Constants.cs†L40-L52】【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Authorization/ServiceBusTokenCredential.cs†L16-L88】

## Azure Messaging Event Hubs

**API endpoint host.** The connection parses the namespace endpoint from the connection string, captures its host as the fully qualified namespace, and retains the full endpoint (including custom emulators) for routing requests.【F:sdk/eventhub/Azure.Messaging.EventHubs/src/EventHubConnection.cs†L158-L210】【F:sdk/eventhub/Azure.Messaging.EventHubs/src/EventHubsConnectionStringProperties.cs†L47-L68】

**AAD scope host.** `EventHubTokenCredential` fixes the default scope host at `https://eventhubs.azure.net`, adding `/.default` when issuing token requests on behalf of user-supplied credentials.【F:sdk/eventhub/Azure.Messaging.EventHubs.Shared/src/Authorization/EventHubTokenCredential.cs†L16-L85】

## Azure Messaging Event Grid

**API endpoint host.** `EventGridPublisherClient` keeps the topic endpoint supplied by the caller (for example, `https://topic.region.eventgrid.azure.net`) in a URI builder and reuses that host for all publish requests.【F:sdk/eventgrid/Azure.Messaging.EventGrid/src/EventGridPublisherClient.cs†L29-L85】

**AAD scope host.** When token credentials are used, the client configures `BearerTokenAuthenticationPolicy` with the fixed audience `https://eventgrid.azure.net/.default` so Entra ID tokens always target the global Event Grid resource.【F:sdk/eventgrid/Azure.Messaging.EventGrid/src/EventGridPublisherClient.cs†L53-L67】

## Summary Table

| SDK | API endpoint host construction | Entra ID scope host |
| --- | --- | --- |
| Azure Storage (Blobs/Queues/Files) | Formats `https://{account}.{service}.core.windows.net` (or user-specified suffix) from the connection string for each service endpoint.【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageConnectionString.cs†L830-L905】【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageConnectionString.cs†L995-L1027】 | Defaults to `https://storage.azure.com/.default`, with service/account audiences adding `/.default`; challenges may supply a resource host that replaces the audience.【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageClientOptions.cs†L21-L99】【F:sdk/storage/Azure.Storage.Blobs/src/Models/BlobAudience.cs†L33-L84】【F:sdk/storage/Azure.Storage.Common/src/Shared/StorageBearerTokenChallengeAuthorizationPolicy.cs†L103-L125】 |
| Azure Data Tables | Builds table endpoints like `https://{account}.table.core.windows.net` (or Cosmos variants) from the connection string and preserves premium hosts.【F:sdk/tables/Azure.Data.Tables/src/TableConnectionString.cs†L480-L528】【F:sdk/tables/Azure.Data.Tables/src/TableServiceClient.cs†L250-L291】【F:sdk/tables/Azure.Data.Tables/src/TableServiceClient.cs†L979-L984】 | Chooses a storage or Cosmos audience via `TableAudience` and appends `/.default`, keeping that scope through challenge flows.【F:sdk/tables/Azure.Data.Tables/src/TableServiceClient.cs†L272-L281】【F:sdk/tables/Azure.Data.Tables/src/TableAudience.cs†L66-L86】【F:sdk/tables/Azure.Data.Tables/src/TableBearerTokenChallengeAuthorizationPolicy.cs†L60-L83】 |
| Azure Data App Configuration | Uses the store endpoint parsed from the connection string (or provided URI) for all requests.【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/ConfigurationClient.cs†L114-L148】【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/ConfigurationClient_private.cs†L37-L63】 | Infers the correct `appconfig` audience from the endpoint suffix and appends `/.default` (unless an explicit audience is set).【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/ConfigurationClientOptions.cs†L73-L89】【F:sdk/appconfiguration/Azure.Data.AppConfiguration/src/AppConfigurationAudience.cs†L10-L57】 |
| Azure Key Vault | Sends requests to the provided `vaultUri`, stored in `KeyVaultPipeline` and exposed as `VaultUri` (e.g., `{vault}.vault.azure.net`).【F:sdk/keyvault/Azure.Security.KeyVault.Keys/src/KeyClient.cs†L40-L82】【F:sdk/keyvault/Azure.Security.KeyVault.Shared/src/KeyVaultPipeline.cs†L17-L59】 | Extracts the challenge’s `resource`/`scope`, enforces host alignment with the vault domain, and uses the resulting host plus `/.default` for token acquisition.【F:sdk/keyvault/Azure.Security.KeyVault.Shared/src/ChallengeBasedAuthenticationPolicy.cs†L111-L178】 |
| Azure Messaging Service Bus | Reads the namespace host from the connection string and exposes the HTTPS service endpoint; helper methods normalize the namespace for SAS signatures.【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Primitives/ServiceBusConnection.cs†L23-L123】【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Administration/ServiceBusAdministrationClient.cs†L2703-L2734】 | Requests tokens for `https://servicebus.azure.net/.default` by default via `ServiceBusTokenCredential`.【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Constants.cs†L40-L52】【F:sdk/servicebus/Azure.Messaging.ServiceBus/src/Authorization/ServiceBusTokenCredential.cs†L16-L88】 |
| Azure Messaging Event Hubs | Captures the namespace endpoint and host from the connection string, supporting emulator endpoints when configured.【F:sdk/eventhub/Azure.Messaging.EventHubs/src/EventHubConnection.cs†L158-L210】【F:sdk/eventhub/Azure.Messaging.EventHubs/src/EventHubsConnectionStringProperties.cs†L47-L68】 | Uses the fixed audience `https://eventhubs.azure.net/.default` for token credentials.【F:sdk/eventhub/Azure.Messaging.EventHubs.Shared/src/Authorization/EventHubTokenCredential.cs†L16-L85】 |
| Azure Messaging Event Grid | Reuses the caller-supplied topic endpoint when issuing publish requests.【F:sdk/eventgrid/Azure.Messaging.EventGrid/src/EventGridPublisherClient.cs†L29-L85】 | Configures Entra ID requests for `https://eventgrid.azure.net/.default`.【F:sdk/eventgrid/Azure.Messaging.EventGrid/src/EventGridPublisherClient.cs†L53-L67】 |
