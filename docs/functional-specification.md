# Functional specification

## Purpose and scope

Precinct is a tenancy-as-a-service platform for applications that need a shared representation of organisations, their members, and user accounts. Each Precinct instance supports a single product. Its associated application manages Precinct organisations and memberships through an HTTP API secured by an API key and reads mirrored account data through that API. Precinct also exposes a webhook endpoint to receive account events from Clerk. It has two primary responsibilities:

1. Manage organisations (tenants) and user membership of those organisations.
2. Maintain a local mirror of Clerk user account data.

Authentication is delegated to third-party authentication providers. Precinct receives account information from those providers; it does not authenticate human users, authenticate passwords, or issue its own authentication credentials. Human users do not access Precinct directly.
    
This document describes the intended capabilities. The [domain model](domain-model.md) describes the concepts and their relationships. Outstanding decisions are tracked in [Specification issues](issues.md); open questions are not yet commitments.

## Concepts

- **Organisation (tenant):** A distinct customer or other tenant whose membership is managed by Precinct.
- **User account:** A local mirror of a Clerk user, used to associate that user with organisations. Clerk owns the user account and its profile data.
- **Membership:** The association between a user account and an organisation. A user may belong to multiple organisations, and an organisation may have multiple users.
- **External identity:** The Clerk user identifier associated with a mirrored account.
- **Client application:** The application associated with the product supported by a Precinct instance. It uses the API to manage or retrieve tenancy and account information.
- **Authentication provider:** An external system that sends account events to Precinct’s webhook endpoint.

## Record fields

Precinct exposes UUID identifiers to the client application for its records. These identifiers remain stable for the lifetime of each record.

An organisation has a stable Precinct UUID, a name, an owner, and optional description and metadata. The owner is an account. The client application controls ownership and determines its meaning.

An account mirror has a stable Precinct UUID, name, and optional email and metadata. Clerk is authoritative for name and email; the client application owns Precinct metadata. Metadata on both record types is opaque JSON: the client application is responsible for its contents, validation, and handling. Roles may be stored in metadata at the application’s discretion; Precinct does not interpret or enforce them. Precinct applies reasonable validation to non-metadata fields: required names must be non-blank, identifiers and ownership references must be valid, and an email, when present, must have a valid email format. Metadata must be valid JSON, but its contents are not interpreted or validated by Precinct. Length limits are measured in Unicode characters:

| Field | Limit |
| --- | --- |
| Organisation name | 1–200 characters |
| Organisation description | Up to 2,000 characters |
| Account name | 1–200 characters |
| Account email | Up to 254 characters |

Required names must contain at least one non-whitespace character. Optional fields may be omitted or null; an email, if supplied, must be non-blank and valid. Metadata may contain any valid JSON value; no application-level field length or structure is imposed on its contents.

## Organisation and membership capabilities

The client application controls organisations through the API. Provider webhooks do not manage organisations.

- Create an organisation and assign it a stable Precinct UUID on creation.
- Transfer ownership through a dedicated API endpoint.
- Mark an organisation pending deletion through the API. After a grace period, a sweep permanently erases the organisation and all its child data.
- Cancel pending deletion before the sweep.
- Retrieve and update an organisation's common details.
- List organisations.
- Add an existing user account to an organisation, retrieve and list its members, and remove a membership.
- Retrieve the organisations associated with a user account.
- Prevent duplicate memberships for the same user and organisation.
- Associate each membership with the correct organisation and user account.

Ownership changes use the dedicated endpoint rather than a general organisation update. The client application has full authority over these operations and determines the product-level effects of pending deletion. The grace period and sweep scheduling remain to be defined. Invitations are out of scope.

## User account capabilities

- Create, update, and delete account mirrors in response to Clerk events; retrieve mirrors through the API. Assign each mirror a stable Precinct UUID on creation.
- Resolve a Clerk user identifier to its account mirror and prevent duplicate mirrors for the same Clerk user.
- Allow the client to manage Precinct-owned account metadata through the API. The API does not create or delete accounts or change Clerk-owned profile fields.
- Support a user account being associated with memberships in multiple organisations.
- Restrict API access to the authorised client application.

Clerk is the source of truth for account existence and profile data. Email addresses are not used to identify or match accounts.

## Deletion lifecycle

An organisation deletion request through the API marks the organisation pending deletion; a sweep after a grace period permanently erases it. Pending organisation deletion may be cancelled before the sweep. Organisation sweeps also erase all organisation child data. A Clerk deletion event marks the corresponding account mirror pending deletion for a later sweep; it does not delete the Clerk account. The grace period, sweep scheduling, account-mirror cancellation behavior, and effects of pending deletion remain to be defined.

## Concurrent updates

Records subject to API updates have a version returned by the API. An update supplies the version the client observed; Precinct applies the update only if it matches the current version, and advances the version atomically with the change. A mismatch is reported as a conflict. Webhook changes also advance the affected account's version. The operations covered and API representation remain to be defined.

## Authentication provider webhooks

The first release supports Clerk as its authentication provider. Precinct exposes a webhook endpoint for Clerk to send account events and verifies each incoming webhook using the configured verification mechanism before processing it. Duplicate deliveries must be handled idempotently. Events may arrive out of order; their ordering and reconciliation are separate from API version checks and remain to be defined. Invalid or unverifiable events must not change account data.

The first release handles Clerk `user.created`, `user.updated`, and `user.deleted` events to create, update, and mark account mirrors pending deletion. The Clerk user ID identifies the mirror and is distinct from the Precinct UUID. Map the user's non-blank first and last names, joined with a space, to `name`; if neither is present, use the username, then the Clerk user ID as a fallback. Map email to the address identified by Clerk's primary email address ID, or null if none is available. Preserve application-owned account metadata rather than overwriting it from Clerk. Apply the same mapping on creation and update, subject to field validation.

The reconciliation strategy and failure and retry responses remain to be defined. Webhook authentication is separate from the API key used by the associated application.

## Authentication and authorisation

Third-party providers authenticate human users for the product. Precinct does not provide a direct interface for those users or authenticate them for API access. The associated application authenticates its HTTP API requests with a single deployment-configured API key; authentication providers authenticate webhook deliveries using their configured verification mechanisms. The client application is the full authority for product-level decisions and end-user authorisation. Precinct imposes no product-level permission or operation constraints on it.

Further provider integrations and webhook operational details remain open decisions.

## Audit logging

Precinct records domain entity lifecycle actions in an audit table in PostgreSQL. These include creation, updates, organisation ownership transfers, pending deletion, cancellation, and deletion sweeps. Account metadata changes from the API and account mirror changes from Clerk webhooks are audited.

Each audit entry records that an action occurred without recording change details or API key secrets. Retention is configurable and defaults to 90 days. Database-backed changes and their audit entries are committed in the same transaction.

## Expected outcomes

- A client can distinguish two organisations and manage their memberships independently.
- A user with memberships in two organisations has one common account and two distinct membership associations.
- A Clerk user identifier resolves to the same Precinct account mirror on subsequent requests.
- Attempts to create duplicate memberships or duplicate account mirrors for a Clerk user are rejected.
- Requests without a valid application API key cannot access the protected API, and unverifiable webhook events cannot change account data.

