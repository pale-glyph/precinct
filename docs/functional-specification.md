# Functional specification

## Purpose and scope

Precinct is a tenancy-as-a-service platform for applications that need a shared representation of organisations, their members, and user accounts. Each Precinct instance supports a single product. Its associated application is responsible for creating, reading, updating, and deleting Precinct records through an HTTP API secured by an API key. Precinct also exposes a webhook endpoint to receive account events from third-party authentication providers. It has two primary responsibilities:

1. Manage organisations (tenants) and user membership of those organisations.
2. Maintain a common model for user accounts.

Authentication is delegated to third-party authentication providers. Precinct receives account information from those providers; it does not authenticate human users, authenticate passwords, or issue its own authentication credentials. Human users do not access Precinct directly.
    
This document describes the intended capabilities. Details listed as open questions are not yet commitments.

## Concepts

- **Organisation (tenant):** A distinct customer or other tenant whose membership is managed by Precinct.
- **User account:** Precinct's common representation of a person across organisations and supported authentication providers.
- **Membership:** The association between a user account and an organisation. A user may belong to multiple organisations, and an organisation may have multiple users.
- **External identity:** An identity asserted by an authentication provider and linked to a Precinct user account. The provider and its subject identifier together identify the external identity.
- **Client application:** The application associated with the product supported by a Precinct instance. It uses the API to manage or retrieve tenancy and account information.
- **Authentication provider:** An external system that sends account events to Precinct’s webhook endpoint.

## Record fields

Precinct exposes UUID identifiers to the client application for its records. These identifiers remain stable for the lifetime of each record.

An organisation has a stable Precinct UUID, a name, an owner, and optional description and metadata. The owner is an account. The client application controls ownership and determines its meaning.

An account has a stable Precinct UUID, name, display name, and optional email and metadata. Metadata on both record types is opaque JSON: the client application is responsible for its contents, validation, and handling. Roles may be stored in metadata at the application’s discretion; Precinct does not interpret or enforce them. Precinct applies reasonable validation to non-metadata fields: required names must be non-blank, identifiers and ownership references must be valid, and an email, when present, must have a valid email format. Metadata must be valid JSON, but its contents are not interpreted or validated by Precinct. Length limits are measured in Unicode characters:

| Field | Limit |
| --- | --- |
| Organisation name | 1–200 characters |
| Organisation description | Up to 2,000 characters |
| Account name | 1–200 characters |
| Account display name | 1–200 characters |
| Account email | Up to 254 characters |

Required names must contain at least one non-whitespace character. Optional fields may be omitted or null; an email, if supplied, must be non-blank and valid. Metadata may contain any valid JSON value; no application-level field length or structure is imposed on its contents.

## Organisation and membership capabilities

The client application controls organisations through the API. Provider webhooks do not manage organisations.

- Create an organisation and assign it a stable Precinct UUID on creation.
- Transfer ownership through a dedicated API endpoint.
- Flag an organisation for deletion through the API; flagging also suspends it. A later sweep permanently erases the organisation and all its child data.
- Remove the deletion flag before the sweep. Removing the flag does not lift the suspension; unsuspending is a separate API operation.
- Retrieve and update an organisation's common details.
- List organisations.
- Add an existing user account to an organisation, retrieve and list its members, and remove a membership.
- Retrieve the organisations associated with a user account.
- Prevent duplicate memberships for the same user and organisation.
- Associate each membership with the correct organisation and user account.

Ownership changes use the dedicated endpoint rather than a general organisation update. The client application has full authority over these operations and determines the effects of suspension. Sweep scheduling remains to be defined. Invitations are out of scope.

## User account capabilities

- Create, provision, update, and delete common user accounts through the API and provider webhooks; retrieve accounts through the API. Assign each account a stable Precinct UUID on creation.
- Link an external identity to a user account and resolve a linked identity to that account.
- Ensure that a given provider and subject identifier cannot be linked to more than one account.
- Support a user account being associated with memberships in multiple organisations.
- Restrict API access to the authorised client application.

API calls and verified provider webhooks are authoritative for account provisioning and lifecycle changes; Precinct imposes no additional provisioning policy. API updates remain subject to the timestamp rule below. Email addresses must not be assumed to uniquely identify or automatically link accounts without an explicit policy.

## Deletion lifecycle

Deletion requests flag the affected record for deletion and suspend it; a later sweep permanently erases it. This applies to deletions initiated through the API or a provider webhook. A deletion flag may be removed before the sweep, but the record remains suspended until separately unsuspended. Organisation sweeps also erase all organisation child data. Sweep scheduling and the effects of suspension remain to be defined.

## Update timestamps

Every API update request must include a timestamp. Precinct checks it against the latest timestamp recorded for the target record and ignores updates with a timestamp equal to or older than the latest recorded timestamp. Successfully applied updates record their timestamp, making repeated updates idempotent. Timestamp format and precision, and the response to an ignored update, remain to be defined.

## Authentication provider webhooks

The first release supports Clerk as its authentication provider. Precinct exposes a webhook endpoint for Clerk to send account events; the integration should permit additional providers in future. It must verify each incoming webhook using the provider's configured verification mechanism before processing it. Verified webhook events apply regardless of the record’s timestamp; the API timestamp comparison does not suppress them. Record the event timestamp as the record’s latest timestamp when supplied, or use the server time at receipt if it is absent. Duplicate deliveries must be handled idempotently. Events may arrive out of order, and their delivery order can affect the resulting state. Invalid or unverifiable events must not change account data.

The first release handles Clerk `user.created`, `user.updated`, and `user.deleted` events to create, update, and flag accounts for deletion and suspension. The Clerk user ID is stored as the external identity and is distinct from the Precinct UUID. Map the user's non-blank first and last names, joined with a space, to `name`; if neither is present, use the username, then the Clerk user ID as a fallback. Map `display_name` to the username when present, otherwise to `name`. Map email to the address identified by Clerk's primary email address ID, or null if none is available. Preserve application-owned account metadata rather than overwriting it from Clerk. Apply the same mapping on creation and update, subject to field validation.

The reconciliation strategy and failure and retry responses remain to be defined. Webhook authentication is separate from the API key used by the associated application.

## Authentication and authorisation

Third-party providers authenticate human users for the product. Precinct does not provide a direct interface for those users or authenticate them for API access. The associated application authenticates its HTTP API requests with an API key that remains valid until revoked; authentication providers authenticate webhook deliveries using their configured verification mechanisms. The client application is the full authority for product-level decisions and end-user authorisation. Precinct imposes no product-level permission or operation constraints on it.

Further provider integrations and webhook operational details remain open decisions.

## Audit logging

Precinct records API key lifecycle actions, including creation, rotation, and revocation, and lifecycle events for domain entities in audit logs. Domain events include creation, updates, and deletion, as well as organisation ownership transfers, suspension and unsuspension, deletion flag changes, and deletion sweeps. Account changes originating from the API or provider webhooks are audited.

Each audit entry is a simple text statement that an event occurred, without recording change details. Entries must not contain API key secrets. Audit logs are split into files by date. Retention is configurable and defaults to 90 days.

## Expected outcomes

- A client can distinguish two organisations and manage their memberships independently.
- A user with memberships in two organisations has one common account and two distinct membership associations.
- A linked external identity resolves to the same Precinct account on subsequent requests.
- Attempts to create duplicate memberships or link an already-linked external identity to another account are rejected.
- Requests without a valid application API key cannot access the protected API, and unverifiable webhook events cannot change account data.

## Open product decisions

