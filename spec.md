<div class="remove">

# Wallet Attached Storage v0.3

**Abstract:** Wallet Attached Storage is a general purpose permissioned
storage API -- CRUD over an HTTP hierarchy with object-capability authorization.

**Status:** Experimental W3C CCG draft, undergoing regular revisions.
Rendered version: https://w3c-ccg.github.io/wallet-attached-storage-spec/

</div>

## Introduction {#introduction}

The Wallet Attached Storage (WAS) specification brings together the lessons
learned from many attempts to standardize permissioned cloud storage over
the years.

This specification aims to provide:

* A set of **design constraints, goals, and requirements** for WAS, see **Appendix
  [[[#goals-and-requirements]]]**.
* A tiered composable **data model** for storage primitives
* An **HTTP API** binding for storage operations (other bindings, such as JSON-RPC
  or CBOR-based RPCs, are left for future work).
* An authorization profile for use with this storage, see
  [[[#was-authorization-profile-v0-1]]].

### Version History

<div class="note">
This subsection is non-normative.

* **v0.1** (January 2025) -- Initial version, created for the MIT Digital
  Credentials Consortium in collaboration with Benjamin Goering, as the
  "Wallet Attached Storage" specification.
* **v0.2** (April 2026) -- Added examples and editorial fixes. Snapshot at
  <https://wallet.storage/spec>.
* **v0.3** (through June 2026) -- Initial incubation at MIT DCC, based on
  implementation experience.
* (upcoming) **v0.4** (mid-July 2026) -- Migrated to the W3C Credentials Community Group
  (CCG) for further incubation. Pending rename to the "Web Spaces API"
  specification.
</div>

### Reading This Document

<div class="note">
This subsection is non-normative. It collects conventions that the rest of the
document relies on, so that a section read in isolation is still intelligible.

**`Authorization: ...` is a placeholder.** Every request example that carries an
`Authorization` header abbreviates a signed [=zCaps=] (capability) invocation
(as opposed to a bearer token). Reads are authorized in WAS just as writes are.
The expanded form -- with the `Digest`, `Capability-Invocation`, and `Signature`
headers -- appears once, in [[[#performing-authorized-api-calls]]].

**A trailing slash means the container.** A path ending in `/` addresses a
container: `GET` lists its members and `POST` adds a member to it. The same path
*without* the trailing slash addresses the item itself: `GET` returns that item's
description, `PUT` creates or replaces it, `DELETE` removes it. So
`/space/{space_id}/{collection_id}/` is the Collection's member list, while
`/space/{space_id}/{collection_id}` is the Collection's description.

**All examples share one Space.** Every example in this document uses the Space
id `81246131-69a4-45ab-9bff-9c946b59cf2e` on the host `example.com`. Path
segments in braces -- `{space_id}`, `{collection_id}`, `{resource_id}` -- are
placeholders for those identifiers.

**Normative lists live in the appendices.** Error `type` URIs are catalogued in
[[[#error-type-registry]]], path segments this specification reserves in
[[[#reserved-path-segment-registry]]], and client-side encryption schemes in
[[[#encryption-scheme-registry]]]. Those registries are normative: they, not the
surrounding prose, are where an implementation looks such values up.
</div>

### Use Cases

Initial use cases that are motivating this work:

* Sharing of W3C Verifiable Credentials from mobile credential wallets, as well
  as document sharing in general
* Serving as standardized storage (and a point of sync and interop) for
  multiple Verifiable Credential wallets.
* Enabling data portability and service provider interoperability for
  user-controlled social networking
* Enabling the "bring your own storage" architecture pattern of web app development
* Providing data storage and authorization frameworks for Agentic AI

### Scope and Conformance Profiles {#scope-and-conformance-profiles}

This specification represents a layered and modular approach to storage, combining
core features and optional extension points.

The layers below describe **conformance tiers**, not the document's reading
order. Each tier adds optional capability on top of the one before it, so an
implementer can stop at any tier and still be conformant. The normative body,
by contrast, is organized container-first (outermost to innermost: Spaces
Repositories, Spaces, Collections, then Resources), which is convenient as a
reference but is the reverse of the tiers. If you're new to the spec, scan the
profile table below for the conformance tiers, then walk through
[[[#quickstart-your-first-request]]] to watch a request succeed. The core tier
is just [[[#resources-and-blobs]]] plus [[[#was-authorization-profile-v0-1]]].

| Profile | Adds | Defining sections |
|---|---|---|
| **Minimal** | Resource CRUD (KV + blob read/write) + authorization | [[[#resources-and-blobs]]], [[[#was-authorization-profile-v0-1]]] |
| **+ Listing** | list resources / collections / spaces | [[[#list-collection-operation]]], [[[#list-all-collections-operation]]], [[[#list-spaces-operation]]] |
| **+ Collection mgmt** | create / manage collections in a Space | [[[#collections]]] |
| **+ Space mgmt** | manage an individual Space | [[[#read-space-operation]]] (Space endpoints) |
| **+ Multi-tenant** | create / manage many Spaces on a server | [[[#spaces-repositories]]] |
| **+ Extensions** | linksets, policy, metadata, export, backends, query, quotas, encryption, replication, versioning | [[[#linksets]]] |

Each tier stacks on the one above it. The **Minimal** profile is a permissioned
key/value (and blob) CRUD API built from simple HTTP verbs and delegatable
capability-based authorization -- enough, on its own, to read and write any
resource (text, structured document, or binary blob) without collection or
space management. Adding **Listing** is what lets you "implement a blog with
comments in 15 minutes". **Collection** and **Space** management will feel
familiar to anyone who has used a GUI front end for a database or file system.
**Multi-tenant** support lets a provider host many Spaces on one server. The
**Extensions** tier layers on optional features -- an access-policy resource,
user-writable metadata (for example, "tags" on binary files), a Space export
endpoint, pluggable [[[#backends]]], query, quotas, client-side encryption (via
[Encrypted Data Vaults](https://identity.foundation/edv-spec/)), replication,
and versioning -- discovered through the linkset feature-detection mechanism
(from [[RFC9264]]; see [[[#linksets]]]).

**Normative status.** Section back-placement does not imply informative status.
Everything from [[[#introduction]]] through [[[#quotas]]] is normative, as are the
appendices [[[#pagination]]], [[[#reserved-path-segment-registry]]],
[[[#encryption-scheme-registry]]], and [[[#error-type-registry]]] (each of which
also carries an inline "This appendix is normative." banner). The remaining
appendices -- [[[#goals-and-requirements]]] and [[[#iana-considerations]]] -- are
informative. Optionality is orthogonal to normativity: many normative sections
describe OPTIONAL endpoint groups, but a server that implements them MUST follow
the stated requirements.

### Quickstart: Your First Request {#quickstart-your-first-request}

<div class="note">
This walkthrough is a non-normative tutorial, not a conformance requirement. It
assumes a server that already hosts a single Space as well as a
`messages` collection, and that you are that Space's [=controller=]. The goal is
to store one JSON Resource and read it back.
</div>

**Step 1: Write.** `PUT` a JSON document to a resource path under a collection.
A `204 No Content` confirms the write:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"hi"}
```

```http
HTTP/1.1 204 No Content
```

**Step 2: Read.** `GET` the same path to retrieve what you just stored:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

```http
HTTP/1.1 200 OK
Content-type: application/json

{"message":"hi"}
```

(The `Authorization: ...` placeholder stands for a signed zCap invocation).
WAS reads are authorized too, so the header is required on the
`GET` just as on the `PUT`. How that credential is constructed is defined in
[[[#performing-authorized-api-calls]]], and its fully expanded form -- with the
`Digest`, `Capability-Invocation`, and `Signature` headers spelled out -- is
shown once in the worked example within that section. Producing that signature
requires a conformant WAS client: it is an Ed25519 `did:key` capability
invocation, not something you can hand-write with curl.

To go beyond a single pre-existing Space -- to create your own Space, or
delegate access to others -- see the conformance-profile table in
[[[#scope-and-conformance-profiles]]] and the full endpoint map in
[[[#api-summary]]].

### API Summary {#api-summary}

API summary at a glance.

Unless otherwise specified by the [=controller=], all **operations
require authorization**. This can be overridden by the controller via the `/policy`
endpoints. Hosting "public-read" resources, such as HTML files for websites,
or media files you can link to via `<img src="">`, is a common use case.

The endpoints below divide into a **Core** profile that every conformant server
implements, and **Optional extensions** grouped by feature. Each row links to the
section that defines its operation. Rows marked **Reserved** name a path this
specification anchors (see [[[#reserved-path-segment-registry]]]) but does not
yet define an operation for; a server MUST NOT repurpose a reserved path.

#### Core

**Resource CRUD (Create, Read, Update, Delete):**

* `POST /space/{space_id}/{collection_id}/` -- [[[#create-resource-add-resource-to-collection-operation]]]
* `GET /space/{space_id}/{collection_id}/{resource_id}` -- [[[#read-resource-operation]]]
* `PUT /space/{space_id}/{collection_id}/{resource_id}` -- [[[#update-or-create-by-id-resource-operation]]]
* `DELETE /space/{space_id}/{collection_id}/{resource_id}` -- [[[#delete-resource-operation]]]
* Use a `HEAD` instead of a `GET` to check a resource's headers/metadata without
  fetching the whole resource. However, do not rely on this for atomic inserts
  (that is, to check if a resource exists before attempting to create it).
  Instead, use the backend-specific Transaction mechanisms when appropriate.

#### Optional extensions

**List resources in a Collection:**

If not implemented on a server, implies that only individual Key/Value operations
are supported.

* `GET /space/{space_id}/{collection_id}/` -- [[[#list-collection-operation]]]

**Manage Collections in a Space:**

If not implemented on a server, implies that collections are pre-configured or
implicit, controlled by the server.

* `POST /space/{space_id}/collections/` -- [[[#create-collection-add-collection-to-a-space-operation]]]
* `GET /space/{space_id}/collections/` -- [[[#list-all-collections-operation]]]
* `GET /space/{space_id}/{collection_id}` -- [[[#get-collection-description-operation]]]
* `PUT /space/{space_id}/{collection_id}` -- [[[#update-or-create-by-id-collection-operation]]]
* `DELETE /space/{space_id}/{collection_id}` -- [[[#delete-collection-operation]]]

**Spaces Repository Endpoints -- Manage Spaces on a Server:**

If not implemented on a server, implies that any existing Spaces are
pre-configured and controlled by the server.

* `POST /spaces/` -- [[[#create-space-operation]]]
* `GET /spaces/` -- [[[#list-spaces-operation]]]

**Space Endpoints -- Manage an individual Space:**

* `GET /space/{space_id}` -- [[[#read-space-operation]]]
* `PUT /space/{space_id}` -- [[[#update-or-create-by-id-space-operation]]]
* `DELETE /space/{space_id}` -- [[[#delete-space-operation]]]
* `POST /space/{space_id}/export` -- **Reserved / not yet specified.** Export
  (download) a Space's contents (all collections and resources).

**Advanced Resource Endpoints:**

* `GET /space/{space_id}/{collection_id}/{resource_id}/meta` -- [[[#read-resource-metadata-operation]]]
* `PUT /space/{space_id}/{collection_id}/{resource_id}/meta` -- [[[#update-resource-metadata-operation]]]

**Policy Related Endpoints:**

Policy overrides are hierarchical and inherited. A policy set for the entire
Space applies to all its Collections and Resources (unless overridden by a
more specific policy, either at the Collection or Resource level).

* `GET|PUT|DELETE /space/{space_id}/policy` -- **Reserved / not yet specified.** CRUD on the policy object for the Space.
* `GET|PUT|DELETE /space/{space_id}/{collection_id}/policy` -- **Reserved / not yet specified.** CRUD on the policy object for the Collection.
* `GET|PUT|DELETE /space/{space_id}/{collection_id}/{resource_id}/policy` -- **Reserved / not yet specified.** CRUD on the policy object for the Resource.

**Linkset / Discovery Endpoints:**

Required if Space endpoints or Collection endpoints are supported.

* `GET /space/{space_id}/linkset` -- [[[#space-linkset]]]
* `GET /space/{space_id}/{collection_id}/linkset` -- [[[#collection-linkset]]]

**Query Endpoints:**

* `POST /space/{space_id}/query` -- **Reserved / not yet specified.** Cross-collection queries (backend-specific).
* `POST /space/{space_id}/{collection_id}/query` -- **Reserved / not yet specified.** Queries within a Collection (backend-specific).

**Backend Management Endpoints** (see [[[#backends]]]):

* `GET /space/{space_id}/backends` -- [[[#space-backends-available]]]
* `GET /space/{space_id}/{collection_id}/backend` -- [[[#collection-backend-selected]]]
  (the backend summary is also displayed in the Collection description object)

**Quota Endpoints** (see [[[#quotas]]]):

* `GET /space/{space_id}/quotas` -- [[[#quotas]]]: the Quota report object, grouped
  by available Backend (add `?include=collections` for a per-collection usage breakdown)
* `GET /space/{space_id}/{collection_id}/quota` -- [[[#quotas]]]: the Quota report
  object for the specific Collection (not all Backends will support per-collection
  quotas however)

## Terminology

<dl class="termlist definitions" data-sort="ascending">
  <dt><dfn data-lt="action|actions|allowedAction">action (allowedAction)</dfn></dt>
  <dd>The kind of operation a request performs on a target, named by a capability
    so it can be authorized. WAS uses the uppercase HTTP method names
    (<a>GET</a>, <a>POST</a>, <a>PUT</a>, <a>DELETE</a>) as its action
    vocabulary. See section
    [[[#authorization-actions-and-the-root-capability]]].</dd>

  <dt><dfn id="term-backend" data-lt="backends">backend</dfn></dt>
  <dd>A storage engine that a [=collection=]'s resources are physically stored
    on, registered at the Space level either by server configuration or by
    client-side "Bring Your Own Storage" registration. Backends are an OPTIONAL
    concern, orthogonal to the Space > Collection > Resource hierarchy: a
    Collection names one via its <code>backend</code> property, and is assigned
    the <code>default</code> backend when it does not. See section
    [[[#backends]]].</dd>

  <dt><dfn data-lt="collections">collection</dfn></dt>
  <dd>A namespace and configuration container for resources. Conceptually maps
    to folders (for file system like storage), buckets (for object storage), or
    database tables (for RDBMSs). See section [[[#collections]]].</dd>

  <dt><dfn data-lt="controllers">controller</dfn></dt>
  <dd>An entity that has the capability to make changes to a given object.</dd>

  <dt><dfn data-lt="did|dids">decentralized identifier (DID)</dfn></dt>
  <dd>See [[DID-CORE]].</dd>

  <dt><dfn data-lt="server|servers|instance|instances">instance, server</dfn></dt>
  <dd>A deployed instance of an application or service that implements this
    specification's API.</dd>

  <dt><dfn data-lt="policies|access control policy|access control policies">policy</dfn></dt>
  <dd>A JSON document with a required <code>type</code> property that declares what
    access a [=target=] grants to callers in general, independent of any [=zCap=]
    a caller might present. A policy is stored at the <code>/policy</code>
    auxiliary resource of a Space, Collection, or Resource. Policies are inherited
    most-specific-wins (Resource over Collection over Space), can only broaden
    access (never deny a caller holding a valid capability), and grant nothing
    when absent or of an unrecognized <code>type</code>. See section
    [[[#access-control-policies]]].</dd>

  <dt><dfn id="term-quota" data-lt="storage quota|storage quotas">quota</dfn></dt>
  <dd>A storage limit enforced per [=backend=], together with the <em>usage</em>
    measurement of the storage consumed against it. Quota reporting and
    enforcement are OPTIONAL and backend-dependent: a Space's per-backend report
    is read from its <code>/quotas</code> auxiliary resource, and a
    per-Collection breakdown from <code>/quota</code>. A write that would exceed
    a quota is rejected with [=quota-exceeded=] (507); a single upload larger
    than the backend's <code>maxUploadBytes</code> constraint is rejected with
    [=payload-too-large=] (413). See section [[[#quotas]]].</dd>

  <dt><dfn data-lt="root capabilities|root zcap">root capability</dfn></dt>
  <dd>The implied capability for a [=target=] whose [=controller=] is the Space's
    controller; it is the root of trust from which all other capabilities for
    that target are delegated. See section [[[#root-capability]]].</dd>

  <dt><dfn data-lt="target|invocationTarget|targets">target (invocationTarget)</dfn></dt>
  <dd>The resource a request acts on -- the full request URL (scheme, host, port,
    and path) -- and the scope a capability authorizes. A capability's
    `invocationTarget` MUST match the request target for the invocation to be
    valid. See section [[[#authorization-actions-and-the-root-capability]]].</dd>

  <dt><dfn data-lt="zcap|zCaps|capability|authorization capability">zCap (Authorization Capability)</dfn></dt>
  <dd>See [zCap Developer Guide](https://interop-alliance.github.io/zcap-developer-guide/) for more
    details.</dd>
</dl>

## Identifiers {#identifiers}

### Identifier Required Properties

Space, Collection, and Resource identifiers used in this specification are
required to have the following properties.

1. **URL-safety** - All characters in a given identifier MUST be URL-safe.
2. **Uniqueness** - All identifiers MUST be unique within a given container.
   That is: Space `id`s (denoted by `{space_id}` in URL templates) MUST be
   unique within a given [=server=], Collection `id`s (denoted by `{collection_id}`
   in URL templates) MUST be unique within a given Space, and Resource `id`s
   (denoted by `{resource_id}` in URL templates) MUST be unique within a given
   Collection.

### Identifier Length and Format

Identifier length limits are currently left to the implementer. However,
implementations SHOULD limit identifier length to their appropriate use case.

Identifier format constraints are currently left to the implementer. Common
identifier formats include:

* Random IDs (either UUIDv4 with hyphens or using a compact encoding).
  Not recommended for _extremely_ high throughput use cases, since
  the clients or servers that are generating them might be limited by available
  device entropy.
* Semi-random Time-sortable IDs (UUIDv7). Useful for cases involving logs,
  histories, feeds, and other situations where sorting by time is desirable.
* Content-based IDs (CIDs). Useful for data deduplication use cases.
* Human-readable IDs. Some use cases might require human-readable identifiers
  (for example, a user hosting a blog in their space might want meaningful collection
  and resource IDs such as `/posts/2020-01-01-hello-world`).

## Authorization

The ability to do cross-domain, operator-independent, standardized cloud storage
operations requires an authorization system that is:

* Modular and layered (for future agility / upgradability)
* Not limited to traditional domain-based "usernames and passwords"
* A hybrid, using object capability principles at its baseline, but also able
  to provide ACL or RBAC-like functionality for user convenience. In other words,
  the system needs to support both "anyone with the link can..." and "these are
  specific people and groups allowed to..." styles of access control
* Compatible with cross-domain replication
* Compatible with end-to-end client side encryption (but also not rely on
  encryption as the sole authorization method)
* "Private by default".
  That is, by default, unless otherwise specified, only the controller of a
  space (or of a collection or resource) is authorized to perform any operation
  (read, write, delete, etc)

As the state of the art in cross-domain authorization advances, we expect there
to be multiple profiles and specs that could be used to perform WAS API
calls. However, to start with, this specification will focus on a single minimal
authorization profile.

### WAS Authorization Profile v0.1 {#was-authorization-profile-v0-1}

Like many authorization specifications, the W.A.S. Authorization Profile tries
to address opposing tensions. On the one hand, to cover the full range of use
cases, it needs to be delegatable, revocable, secure, flexible, and thus
capability based. On the other hand, for ease of implementation and adoption,
and for maximum developer usability, the profile must make the most common
operations as simple and friction free as possible.

To that end, the profile offers the following layered mechanisms.

1. **Root Access**: For basic admin CRUD operations, use the space's `controller`
   DID directly to sign API calls with HTTP Signatures.
2. **Public Read**: For the common "public read" use case (the typical web
   publishing workflow, where a site or a file is shared for anyone to access
   via an HTTP GET), use the simple `{ "type": "PublicCanRead" }` WAS
   Authorization syntax, see below.
3. **Advanced Delegatable Capabilities** ("anyone with the link..." style):
   Use zCaps [Authorization Capabilities v0.3](https://w3c-ccg.github.io/zcap-spec/)
4. **Policy Based Access Control** (including the familiar "share with this list
   of people or groups" style): Use the space's `linkset` property to point to
   a linkset that includes a URL to an access control policy document.

#### Authorization Specification Dependencies at a Glance

The initial W.A.S. Authorization Profile uses the following specifications.

1. Identity (for controllers or clients/agents): [DID 1.0](https://www.w3.org/TR/did-1.0/)
2. Capability data model: [Authorization Capabilities for Linked Data v0.3](https://w3c-ccg.github.io/zcap-spec/)
3. Protocol for obtaining authorization: Out of scope (implementers are encouraged
   to use VC-API, OpenId4VP, OAuth2, or GNAP, as appropriate)
4. Proof of Possession / authorization invocation: HTTP Signatures.
   MUST - [RFC 9421 HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html),
   MAY - [HTTP Signatures (Cavage draft 12)](https://datatracker.ietf.org/doc/html/draft-cavage-http-signatures)
5. Request body integrity: the `Digest` header, bound to the request
   signature -- see [[[#request-body-integrity-digest-header]]]
6. Access Control / Policy language data model: see
   [[[#access-control-policies]]] (`PublicCanRead` is the only normative type for
   v0.1)

#### Space `controller` and the Root of Trust {#space-controller-and-the-root-of-trust}

Conceptually, the space's controller serves as the root of trust and authorization
for any operations on the space or its collections or resources.
That is, any operation requiring an authorization MUST provide a chain of proof
all the way to the space controller, by one of the following:

1. Direct: Provide a root capability invoked directly by the controller, or
2. Delegated: Invoke a capability delegated to some other agent by the controller, or
3. Matching Policy: (if using any kind of access control policy mechanism) Match
   an authorization policy specified in the `linkset` property of the space. This
   resource is related to the space controller because initially, it can only be
   modified either by the controller or an authorized party delegated to by the
   controller.

Space `controller`s MUST be in the form of a [DID](https://www.w3.org/TR/did-1.0/).

For minimal compatibility, all WAS implementations MUST support the
[`did:key` DID Method](https://w3c-ccg.github.io/did-key-spec/), using the
Multikey encoding of `Ed25519` elliptic curve keys, as specified in the
[Multikey section of the CID spec](https://www.w3.org/TR/cid-1.0/#Multikey)
as the space `controller`.

When a space is created via an HTTP `POST` or `PUT` operation (see
[[[#http-api-post-spaces]]] and [[[#http-api-put-space-space_id]]]), the
controller for that space
is set explicitly. That is, a client specifies the `controller` as part of the
payload of the PUT or POST create space request, and the server MUST verify
that the invocation is authorized by that `controller`, by one of the first two
mechanisms above: either directly -- the signing key (key ID) used in the
headers is authorized in the `capabilityInvocation` section of the
`controller`'s DID document -- or via a capability delegated by the
`controller` to the signing DID. (The third mechanism, matching policy, does
not apply: no Space, and therefore no policy, exists yet.)

See [[[#http-api-post-spaces]]] below for examples of `controller`
determination and verification.

#### Performing Authorized API Calls {#performing-authorized-api-calls}

Unless otherwise explicitly allowed via access control policy (see below),
all W.A.S. API calls require authorization.

This can be done in one of two ways:

1. (for admin-like root access) Use the `controller` DID directly to sign
   HTTP API requests using the HTTP Signatures specification, invoking the
   target's [=root capability=].
2. (for advanced delegatable use cases) Use HTTP Signatures in combination
   with [Authorization Capabilities v0.3](https://w3c-ccg.github.io/zcap-spec/),
   and include a capability invocation header in the API request.

Throughout this specification, the request examples abbreviate this header as a
placeholder, `Authorization: ...`, rather than reproducing a full signature or
capability invocation. In each case it stands for a credential constructed as
described in [[[#performing-authorized-api-calls]]].

#### Request Body Integrity (Digest Header) {#request-body-integrity-digest-header}

When an authorized request carries a body (a Resource write, a Space create,
and so on), this profile binds the body to the request's HTTP Signature so
that the payload cannot be substituted without invalidating the signature:

1. The client MUST include a `Digest` header whose value is the hash of the
   request body, carried as a `mh` (multihash) parameter: a multibase
   base64url-encoded (`u` prefix) multihash of the body's SHA-256 digest
   (Multihash and Multibase as defined in the
   [CID 1.0 specification](https://www.w3.org/TR/cid-1.0/)). For example:

   ```http
   Digest: mh=uEiCPO-qYr-z0GYV5F75-N1l8Rhjv4xIkKZsnbTZeZ7emSA
   ```

2. The `content-type` and `digest` headers MUST be included in the
   signature's covered (signed) headers list, alongside `(key-id)`,
   `(created)`, `(expires)`, `(request-target)`, `host`, and
   `capability-invocation`.
3. For any request that carries a `Content-Type` header, the server MUST
   require `digest` among the covered headers, and SHOULD independently
   recompute the digest of the received body and compare it to the `Digest`
   header value. A missing, malformed, or non-matching `Digest` on a request
   with a body is rejected with an [=invalid-authorization-header=] (400)
   error.

Bodyless requests (`GET`, `HEAD`, `DELETE`) carry no `Digest` header.

Example authorized write request, showing the `Digest` header and the
covered headers list (the space's [=controller=] invoking the
[=root capability=] for the target; line breaks within the `Authorization`
header are for display only):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/photos/sunset.png HTTP/1.1
Host: example.com
Content-Type: image/png
Digest: mh=uEiCPO-qYr-z0GYV5F75-N1l8Rhjv4xIkKZsnbTZeZ7emSA
Capability-Invocation: zcap id="urn:zcap:root:https%3A%2F%2Fexample.com%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e%2Fphotos%2Fsunset.png",action="PUT"
Authorization: Signature keyId="did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW#z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  headers="(key-id) (created) (expires) (request-target) host capability-invocation content-type digest",
  signature="6GoRQ+rW69wBhNyERkafAXEZXZArezHvGRNUWC0HNI4Ss1xAiiMHdayS5aA2R6hLuYRNw6h9J9eCmQVMuHE1Bw==",
  created="1758150502",expires="1758151102"

...binary PNG bytes...
```

<div class="ednote">
**Digest vs Content-Digest.** The `Digest` header used by this profile
descends from [[RFC3230]] (Instance Digests in HTTP). [[RFC9530]] (Digest
Fields) obsoletes RFC 3230 and replaces `Digest` with `Content-Digest` /
`Repr-Digest`. The current WAS implementation stack uses the legacy header
with a multihash value; migration to `Content-Digest` (alongside the move to
[[RFC9421]] HTTP Message Signatures) is a future direction for this profile.
</div>

#### Authorization Actions and the Root Capability {#authorization-actions-and-the-root-capability}

A capability invocation names an [=action=] that the invoked capability must
permit. WAS uses the uppercase HTTP method names as its action vocabulary:

* <dfn id="get-action">`GET`</dfn> -- read a Space, Collection, or Resource. A
  `HEAD` request is authorized as a `GET`.
* <dfn id="post-action">`POST`</dfn> -- create a child item in a container (add a
  Resource to a Collection, a Collection to a Space, or a Space to the Spaces
  Repository).
* <dfn id="put-action">`PUT`</dfn> -- create-by-id or replace a Space,
  Collection, or Resource.
* <dfn id="delete-action">`DELETE`</dfn> -- delete a Space, Collection, or
  Resource.

A request is authorized by a capability when all the following hold:

1. the capability's `invocationTarget` matches the request's [=target=] -- the
   full request URL (scheme, host, port, and path);
2. the capability's `allowedAction` includes the request's action (the HTTP
   method); and
3. the invocation is signed by a key the capability authorizes, carried as a
   valid HTTP Signature over the request (see
   [[[#performing-authorized-api-calls]]]).

##### Root Capability {#root-capability}

Every [=target=] has an implied **root capability** whose `controller` is the
Space's [=controller=]. It is identified by the URI `urn:zcap:root:` followed by
the percent-encoded target URL:

```json
{
  "@context": "https://w3id.org/zcap/v1",
  "id": "urn:zcap:root:https%3A%2F%2Fexample.com%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e%2Fmessages%2Fhello-world",
  "invocationTarget": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

The Space [=controller=] MAY invoke the root capability directly -- signing the
request with a key listed in the `capabilityInvocation` section of the
controller's DID document -- to perform any operation. This is the "root access"
path. All other authorized access derives from a capability delegated, directly
or transitively, from this root.

##### Delegation {#delegation}

To grant another agent access, the [=controller=] (or any agent holding a
sufficiently broad capability) delegates a capability that names the grantee as
its new `controller`, the `invocationTarget` to scope it to, and the
`allowedAction`s to permit. A delegation MAY set an `expires` time. For example,
granting another DID read-only access to a single Collection:

```json
{
  "@context": "https://w3id.org/zcap/v1",
  "id": "urn:uuid:6c9f3a1e-2b4d-4f8a-9c1e-7d2b3a4c5e6f",
  "parentCapability": "urn:zcap:root:https%3A%2F%2Fexample.com%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e%2Fmessages%2F",
  "invocationTarget": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/",
  "controller": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "allowedAction": ["GET"],
  "expires": "2026-12-31T23:59:59Z",
  "proof": { "...": "delegation proof signed by the parent capability's controller" }
}
```

The delegated capability is handed to the recipient out of band. The recipient
invokes it by signing a request with their own key and including the capability
in the `Capability-Invocation` header.

#### Specifying Access Policy With Space Link Sets

To set access control policy for a space, use the `linkset` property.

Example (fetching a space's link set):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset HTTP/1.1
Host: example.com
Accept: application/linkset+json
Authorization: ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
      "https://wallet.storage/spec#policy": [
        { 
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/policy",
          "type": "application/json"
        }
      ]
    }
  ]
}
```

Example (fetching a specific policy document from the link set):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/policy HTTP/1.1
Host: example.com
Authorization: ...
```

```http
HTTP/1.1 200 OK
Content-type: application/json

{ "type": "PublicCanRead" }
```

#### Access Control Policies {#access-control-policies}

Capabilities answer the question "does the caller hold a credential that grants
this action?" Access control *policies* answer the complementary question
"does this target grant this action to callers in general (or to a named set of
principals)?" Policies are how a [=controller=] makes a target public-readable
(or, in future profiles, shares it with a list of people or groups) without
having to issue a capability to each caller.

A policy is a JSON document with a required `type` property, stored at the
`/policy` auxiliary resource of a Space, Collection, or Resource and
discoverable via the [=policy relation=] in the linkset (see [[[#space-linkset]]] and
[[[#collection-linkset]]]).

**Evaluation contract:**

* **Capability first, policy second.** A request is first checked against any
  capability invocation it carries. The effective policy is consulted only as a
  fallback, and it can only **broaden** access -- a policy never denies a caller
  who presents a valid capability.
* **Fail-closed.** An absent policy, or a policy whose `type` an implementation
  does not recognize, grants nothing.
* **Most-specific-wins inheritance.** The effective policy for a target is the
  one set at the most specific level that has a policy document: a Resource
  policy overrides a Collection policy, which overrides a Space policy.
* **Access kind.** For policy evaluation, the request action is reduced to a
  coarse access kind: [=GET=] (and `HEAD`) is a `read`; [=POST=], [=PUT=], and
  [=DELETE=] are a `write`.

##### `PublicCanRead`

For v0.1, the only normative policy `type` is `PublicCanRead`:

```json
{ "type": "PublicCanRead" }
```

It grants the `read` access kind to any caller (including unauthenticated ones)
and grants no write access. This is the canonical "public read" pattern -- for
example, hosting an HTML file or an image that anyone may `GET`, while writes
still require a capability. Setting it on a Space makes the whole Space
public-readable (subject to any more specific Collection or Resource policy);
setting it on a single Resource exposes only that Resource.

## Error Handling {#error-handling}

This specification uses [[RFC9457]] Problem Details for HTTP APIs for error responses.

* Error responses SHOULD be returned using the `application/problem+json` content type.
* `type` and `title` properties are REQUIRED.
* `errors` array with `{ detail, pointer }` objects is encouraged where appropriate.

The `type` property is a URI identifying the _kind_ of problem (not the
operation), and the same `type` is reused across operations. See Appendix
[[[#error-type-registry]]] for the catalog of `type` URIs this specification
defines, along with their typical status codes and a canonical example response
for each kind.

When returning errors, keep in mind the principle of **maximum privacy**:
always "not found" instead of "not authorized". The _existence_ of a resource,
collection, or space, is by itself an item of sensitive information.
If a client makes an API call, and they have insufficient authorization to
perform that action, a "not found" error (such as HTTP 404) MUST be returned,
just as if that resource (or space or collection) did not exist.
To put it another way, an unauthorized client (meaning, either not carrying
any authorization in the request itself, or possessing insufficient permissions)
MUST NOT be able to discover the existence of a resource based on the error
response.

This privacy rule governs failures that would otherwise reveal whether a target
*exists*. It does not require masking failures that describe the *request
itself*:

* **Request and credential validation** failures: a malformed or missing
  request body, a missing `Content-Type`, or a capability invocation or
  signature that is present but malformed or fails to verify -- MAY be reported
  with precise status codes and `type`s (for example [=invalid-request-body=],
  [=missing-content-type=], [=invalid-authorization-header=], or
  [=controller-mismatch=]). These responses describe the request and do not
  disclose whether any particular target exists.
* **Authorization** failures against an existing target: a well-formed request
  that simply lacks sufficient privilege, including one that carries no
  authorization at all, MUST be reported as [=not-found=] (HTTP 404),
  indistinguishable from the target being absent.

"List Spaces" operations are an exception to 404 masking: rather than returning
an error, they return `200 OK` with only the subset of items the caller is
authorized to see (an empty `items` array if none) -- see [[[#list-spaces-operation]]].

## Common Behaviors

This section defines HTTP mechanics that are shared across the operations in
this specification, rather than belonging to any single endpoint: caching,
conditional requests (optimistic concurrency control), and pagination of list
responses.

### Caching

WAS relies on ordinary [[RFC9111]] HTTP caching and defines no caching layer of
its own. On the read side, a Resource's strong `ETag` validator (see
[[[#conditional-requests]]]) drives standard validation: a `GET` carrying
`If-None-Match: "<etag>"` yields `304 Not Modified` when the Resource is
unchanged. Servers SHOULD emit `ETag` (and MAY emit `Last-Modified`) on
`GET`/`HEAD` responses, and SHOULD mark responses to non-idempotent operations
as non-cacheable (for example, `Cache-Control: no-store`).

<div class="ednote">
Freshness lifetime and `Cache-Control` directive semantics beyond validation
are not yet specified.
</div>

### Conditional Requests {#conditional-requests}

Servers and backends MAY support [[RFC9110]] conditional requests to provide
optimistic concurrency control on writes -- the mechanism that prevents the
"lost update" problem, where two clients that both read version N of a Resource
each write version N+1 and the second silently clobbers the first. A backend
that supports this advertises the `conditional-writes` feature in its Backend
description (see [[[#backend-data-model]]]); a client SHOULD use these
preconditions only against a backend that advertises support.

When supported, a Resource carries a strong **`ETag`** validator that changes
whenever its stored content changes. Servers SHOULD return the `ETag` on
`GET`/`HEAD` responses. The validator is opaque to clients: how a backend
derives it is a server-side concern (see [[[#backend-data-model]]]).

A state-changing request (`PUT` or `DELETE`) MAY carry a precondition:

* **`If-Match: "<etag>"`** -- perform the write only if the Resource's current
  `ETag` matches it (an "update-if-unchanged"). If it does not match, the server
  MUST NOT perform the write and MUST respond with [=precondition-failed=]
  (`412`).
* **`If-None-Match: *`** -- perform the write only if the Resource does not yet
  exist (a "create-if-absent"). If it already exists, the server MUST NOT
  perform the write and MUST respond with [=precondition-failed=] (`412`).

A server that supports conditional writes MUST evaluate the precondition
atomically with the write, so that two concurrent writers cannot both observe
the same prior version and both succeed; how a backend achieves this atomicity
is a server-side concern (see [[[#backend-data-model]]]). A client recovers
from a `412` by re-reading the current Resource, re-applying its change on top
of the new version, and retrying.

As with [=id-conflict=], a server MUST verify the caller's authorization
before evaluating a precondition, so a `412` is only ever observed by a caller
already authorized to write the target; an under-authorized caller receives the
merged [=not-found=] (`404`) instead, per the maximum-privacy rule in
[[[#error-handling]]].

A `412` arises only from an explicit `If-Match` / `If-None-Match`
precondition header. It is deliberately distinct from the header-less `409`
conflict kinds -- [=id-conflict=] (a `POST` create whose chosen `id` is already
taken) and [=reserved-id=] -- which describe a conflict with current state where
the client stated no precondition. Conditional requests are the versioned,
header-driven concurrency mechanism; the `409` kinds are not.

### Paginated List Responses {#paginated-list-responses}

The list operations -- [[[#list-spaces-operation]]], [[[#list-all-collections-operation]]],
and [[[#list-collection-operation]]] -- MAY paginate their responses, returning
one page of items at a time using the cursor-based profile defined in Appendix
[[[#pagination]]]. Pagination is OPTIONAL: a server that returns every item in
a single response is conformant, and a client MUST be prepared for either
behavior.

## Spaces Repositories {#spaces-repositories}

A Spaces Repository is a set of API endpoints that supports the creation and
management of multiple spaces on a given [=server=].
This `/spaces/` set of API endpoints is optional. If a server does not support
this feature (for example, if it is a single-tenant server with an existing
hardcoded Space), then it can implement only the `/space/{space_id}/` endpoints
and get most of the functionality of this specification.

<div class="ednote">
The Spaces Repository endpoints are a candidate for extraction into a
standalone companion specification (multi-tenant space provisioning) in a
future version of this document.
</div>

### Create Space operation {#create-space-operation}

To create a Space:

* Perform an authorized Create Space operation that includes a Proof of
  (cryptographic material) Possession via a mechanism such as HTTP Signatures.

* If no `id` is provided, it will be generated by the storage server.

* A `controller` DID MUST be provided in the request body, and the request MUST
  be authorized by that DID.
  * If no `controller` is provided, the server MUST return an HTTP 400 error
    response
  * The capability invocation MUST be *authorized by* the body's `controller`:
    either signed directly by the `controller` DID (the common case), or signed
    by another DID presenting a valid, unexpired delegation chain rooted in the
    `controller` (see [[[#delegation]]]). This is how the root of trust is
    initially set up (see [[[#space-controller-and-the-root-of-trust]]] for
    more details).
  * The delegated form supports creating a Space *on behalf of* its eventual
    controller: for example, a provisioning service that handles payment or
    onboarding can be delegated a `POST` capability for `/spaces/` by the user,
    and create a Space that the user's DID controls from the start. What is
    never acceptable is installing a `controller` that has authorized nothing
    at all: that would let anyone bind arbitrary DIDs to Spaces without their
    consent -- squatting meaningful ids "in their name", or burning
    per-controller onboarding allowances such as "one free Space per
    controller".

* (Optional, out of scope) A given storage provider MAY impose additional
  requirements in order to create a Space for a given controller, such as:
  - a Verifiable Credential representing a pre-arranged onboarding coupon
  - a proof of payment
  - a proof of membership in an organization

#### (HTTP API) POST `/spaces/` {#http-api-post-spaces}

To create a space via HTTP API using the Spaces Repository POST API:

```http
POST /spaces/ HTTP/1.1
Host: example.com
Accept: application/json
Content-type: application/json
Authorization: ...

{
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Example success response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Note that in the example above:

* the `id` was not specified in the body of the request, and so was generated by
  the server and returned in the response

#### Create Space Errors {#create-space-errors}

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=invalid-request-body=] (400) -- the required `controller` property is
  missing from the request body.
* [=missing-authorization=] (401) -- the request lacks a valid proof of
  possession of the `controller` DID.
* [=controller-mismatch=] (400) -- the invocation is not *currently*
  authorized by the `controller` DID in the request body: it is neither signed
  by that DID nor accompanied by a valid, unexpired delegation chain rooted in
  it. A chain rooted in a different DID, an expired delegation, and a chain
  whose proof fails verification all fall under this one type; servers are not
  required to distinguish them (delegation-chain verifiers often report
  failure opaquely), but SHOULD differentiate the cause in the non-normative
  `detail` string where they can -- in a delegated provisioning flow, the
  cause determines who must act (the user re-delegates an expired capability;
  the service corrects a request whose chain is rooted in the wrong DID).
* [=invalid-id=] (400) -- the supplied Space `id` is not URL-safe (see
  [[[#identifiers]]]).
* [=id-conflict=] (409) -- a Space with the supplied `id` already exists.

Onboarding requirements, if any, are provider-specific and out of scope for this
specification (see the optional onboarding material above). Their error `type`,
`title`, and status code are likewise provider-defined.

A `POST /spaces/` that supplies an `id` already in use returns the
[=id-conflict=] error. To create or replace a Space at a client-chosen
`id` without conflict, use the idempotent
[[[#update-or-create-by-id-space-operation]]] instead.

The existence check behind [=id-conflict=] is security-critical here, not
merely a conformance detail. Create Space is the one operation whose capability
invocation is verified against the `controller` supplied *in the request
body* -- the Space does not exist yet, so there is no stored controller to
verify against. A server that instead treats `POST /spaces/` as
create-or-replace turns this operation into a takeover: any caller able to
construct a valid invocation for their *own* `controller` could overwrite an
existing Space's description -- controller included -- simply by POSTing its
`id`. Servers MUST check for an existing Space with the supplied `id`, and
reject with [=id-conflict=], before writing anything.

<div class="note">
The [=id-conflict=] response on this operation necessarily reveals whether a
Space id is taken: any caller permitted to attempt creation can distinguish
`409` from `201` -- just as they could by observing whether creation succeeds.
This disclosure is inherent to client-chosen ids on a create endpoint, and is
not a violation of the maximum-privacy principle of [[[#error-handling]]],
which governs errors about *existing* targets the caller is not authorized
for. Unguessable (for example, UUID) Space ids keep the signal worthless to an
attacker, and providers that gate Space creation behind onboarding requirements
also bound who can observe it.
</div>

<div class="note">
Differentiated `detail` strings on [=controller-mismatch=] are likewise
privacy-safe: Create Space has no existing target to protect, and everything
its verification examines -- the body's `controller`, the capability chain,
its signatures -- is supplied by the caller, so failure granularity reveals
nothing the caller does not already hold. This reasoning does *not* extend to
failure causes that depend on server-side state (for example, capability
revocation status or per-controller onboarding allowances); whether and how to
disclose those remains provider-defined.
</div>

### List Spaces Operation {#list-spaces-operation}

* Requires appropriate authorization (root zcap invoked by the controller of one
  or more spaces, or a zcap granting permission to read a one or more spaces)

* Lists only the spaces the requester is authorized to see

* MAY be paginated (see [[[#pagination]]])

#### (HTTP API) GET `/spaces/`

Example request:

```http
GET /spaces/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response (requester has read access to at least one space):

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "url": "/spaces/",
  "totalItems": 1,
  "items": [
    {
      "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e"
    }
  ]
}
```

Example success response (requester does NOT have access to any spaces):

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "url": "/spaces/",
  "totalItems": 0,
  "items": []
}
```

A request that carries no authorization (or authorization for no spaces) is not
an error: like any list operation it returns the `200 OK` empty-list response
shown above, revealing nothing about which spaces exist (see
[[[#error-handling]]]).

## Spaces {#spaces}

A space is a namespace for collections and a unit of general configuration,
a volume of storage that contains one or more collections.
Conceptually, is maps to a disk partition (for file systems), or a database
(for relational databases).

### Space Data Model {#space-data-model}

`Space` properties:

* `id` - A unique identifier for a space at a given [=server=]. Created by the
  server if not provided. Note: the `{space_id}` template parameter used in URL
  templates in this spec MUST match that space's `id` property. See
  [[[#identifiers]]] for additional constraints.
* `type` - An array of strings, MUST include the type `Space`. The array SHOULD
  be lexically sorted so that the object has a canonical, stable serialization
  (useful for hashing or signing).
* `name` (optional) - An arbitrary human-readable name for the space. Does not
  have to be unique.
* `controller` - A cryptographic identifier (a [=did=])
  of the entity that is authorized to perform operations on the space (or to
  delegate authorization to other entities)

Space properties automatically added by the server:

* `url` - A relative URL to the space's description resource.
  Added by the server, used in the Space Description object as well as
  the [[[#list-spaces-operation]]] result.
* `linkset` - A relative URL to a resource which contains
  a set of links to auxiliary resources (such as to access control policy
  documents). See section [[[#space-linkset]]].
  Note that this is one of the [[[#space-level-reserved-endpoints]]].

### Read Space operation {#read-space-operation}

* Requires appropriate authorization (root zcap invoked by the space's controller,
  or a zcap granting permission to read a particular space)
* Returns the details for the specified space `id`
* Only includes the resources the requester is authorized to see

The format of the response is determined based on content negotiation;
`application/json` is the REQUIRED baseline (see
[[[#content-types-and-representations]]]).

#### (HTTP API) GET `/space/{space_id}`

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset"
}
```

#### Read Space Errors

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space does not exist, or the caller has missing
  or insufficient authorization. A server MUST return the same error response in
  both cases, per [[[#error-handling]]].

### Update (or Create by Id) Space operation {#update-or-create-by-id-space-operation}

When creating or modifying a Space via PUT, the client specifies the `id`
of the Space.

* Requires appropriate authorization (root zcap invoked by the space's
  controller, or a zcap granting permission to write to a particular space)
* Allows the client to update the following fields:
  - `name`
  - `controller`

The `controller` property in the request body is the *proposed* value, not the
root of trust for the request. When the Space already exists, the capability
invocation MUST be verified against the Space's currently *stored*
`controller`: only the current controller (or its delegate) may update the
Space, including transferring it by writing a new `controller`. The body's
`controller` participates in verification only when the `PUT` creates the
Space -- there is no stored controller yet, so, as with
[[[#create-space-operation]]], the invocation MUST be authorized by the body's
`controller`: signed by it directly, or presented with a valid, unexpired
delegation chain rooted in it (violations are [=controller-mismatch=], as for
POST). A server that verifies an *update* against the
body's `controller` reopens the takeover described under
[[[#create-space-errors]]]: any caller could seize an existing Space by
PUTting its `id` with themselves as the `controller`.

#### (HTTP API) PUT `/space/{space_id}`

Note that this is a _full_ update (partial updates via http `PATCH` verb might
be supported later). However, some fields may not be updated (like `id`) and so
may be omitted from the request payload.

Note that this operation is idempotent.

* When creating a space via PUT, a `controller` property is required in the PUT request body.

Example request (creating a new space via PUT), note the lack of trailing slash:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e HTTP/1.1
Host: example.com
Accept: application/json
Content-type: application/json
Authorization: ...

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Example success response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e
```

Example request (updating the `name` property of a space). Note that
server-managed properties such as `url` and `linkset` are not client-writable
and are omitted from the request body:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e HTTP/1.1
Host: example.com
Content-type: application/json
Accept: application/json
Authorization: ...

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Newly renamed space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space does not exist, or the caller has missing
  or insufficient authorization; per [[[#error-handling]]] the two are
  indistinguishable.
* [=invalid-request-body=] (400) -- the client is attempting to change an
  immutable field, such as setting a body `id` that does not match the
  `{space_id}` in the request URL.

### Delete Space operation {#delete-space-operation}

* Requires appropriate authorization (root zcap invoked by the space's
  controller, or a zcap granting permission to write to a particular space)
* Deletes the space and all the data (collections and resources) contained
  in it
* This operation is idempotent

#### (HTTP API) DELETE `/space/{space_id}`

Example request (no request body):

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the caller has missing or insufficient authorization.
  Because DELETE is idempotent, an authorized request for an already-absent
  Space returns `204`; an under-authorized request returns `404` instead, per
  [[[#error-handling]]], so that an unauthorized caller cannot probe for
  existence.
* [=invalid-id=] (400) -- the supplied Space `id` is not URL-safe.

### List All Collections operation {#list-all-collections-operation}

* Returns the list of all Collections in a Space (that the requester has
  permission to access)
* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request must be
    signed by the space's [=controller=], or invoke a delegated capability that
    allows the [=GET=] action
* Since Collection's `name` property is optional, default it to be the same
  value as `id` when `name` is missing. (The name is intended to drive UIs, so
  defaulting to `id` simplifies consuming client logic.)
* MAY be paginated (see [[[#pagination]]])

#### (HTTP API) GET `/space/{space_id}/collections/`

Example request (note the trailing slash):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/collections/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/collections/",
  "totalItems": 2,
  "items": [
    {
      "id": "example",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/example",
      "name": "Example Collection"
    },
    {
      "id": "73WakrfVbNJBaAmhQtEeDv",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
      "name": "73WakrfVbNJBaAmhQtEeDv"
    }
  ]
}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space does not exist, or the caller has missing or
  insufficient authorization; per [[[#error-handling]]] a Space the caller is
  not authorized to read is indistinguishable from one that does not exist.

## Collections {#collections}

A collection is a namespace for Resources, and a unit of configuration, within
a space.

In other storage systems, the concept of collections has many different names.
For example, _Directory, Folder, RDBMS Table, Document Collection, Graph, WebAPI
FileList, Bucket, LDP Basic Container, EDV Vault_, and so on.

Collections do not contain other collections (this specification adopts a
flat collection structure).

<div class="note">
Rationale: Nested collections (such as those used by Solid/LDP/LWS) encode query
relationships into URL hierarchy, which creates significant complexity
(recursive permissions, recursive deletes, path traversal, depth-limited listing)
and maps poorly to flat backends like RDBMS tables or S3 buckets. The use cases
that motivate nesting (e.g., "all comments on a post") are better served by the
`query` endpoint with field-based filtering.
</div>

### Collection Data Model {#collection-data-model}

Collection properties (user-writable):

* `id` - A unique collection identifier (within a given space). Created by the
  server if not provided. Note: the `{collection_id}` template parameter used in
  URL templates in this spec MUST match that collection's `id` property.
  See [[[#identifiers]]] for additional constraints.
* `type` - An array of strings, MUST include the type `Collection`. As with a
  Space's `type`, the array SHOULD be lexically sorted for a canonical, stable
  serialization.
* `name` (optional) - An arbitrary human-readable name for the collection. Does not
  have to be unique.
* `backend` (optional) - An object describing the storage backend selected for
  this collection. If not specified, defaults to the value `{ "id": "default" }`.
  The backend object's `id` property MUST be from the list of [[[#space-backends-available]]]
  for the given space. If an unavailable (unsupported) backend is specified,
  the server MUST throw an error.
  See section [[[#backends]]] for more details.
* `encryption` (optional) - A non-secret marker declaring that this collection's
  Resources are client-side encrypted, and naming the scheme. An object with a
  required string `scheme` property (e.g. `{ "scheme": "edv" }` for the
  EDV-over-WAS scheme); absent means the collection is plaintext. The server
  MUST NOT interpret the marker beyond validating its shape -- it never holds key
  material and stores the marker opaquely. Its purpose is discovery: any
  authorized reader (including a delegated consumer that did not create the
  collection) learns from the Collection Description that the collection is
  encrypted, and decrypts with its own keys. The marker is **set-once**: a server
  MUST allow declaring it on a collection that lacks one (e.g. migrating a
  pre-existing collection), but MUST reject (with an `encryption-immutable` error)
  any attempt to change its `scheme` or clear it on an existing collection, since
  that would corrupt the already-stored encrypted Resources. See section
  [[[#backends]]] (client-side encryption note) for the rationale.
  A server that recognizes the declared `scheme` enforces it structurally on
  write -- rejecting any non-envelope body so plaintext can never be stored in
  an encrypted Collection; see [[[#encryption-scheme-registry]]].

Collection properties automatically added by the server:

* `url` - A relative URL to the collection's description resource.
  Added by the server, used in the Collection Description object as well as
  the List Collections operations result.
* `linkset` - A relative URL to a resource which contains
  a set of links to auxiliary resources (such as to access control policy
  documents). See section [[[#collection-linkset]]].
  Note that this is one of the [[[#collection-level-reserved-endpoints]]].

Example collection (JSON representation):

```json
{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
  "type": ["Collection"],
  "name": "Verifiable Credentials Collection",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/linkset"
}
```

### Create Collection (Add Collection to a Space) operation {#create-collection-add-collection-to-a-space-operation}

When a Collection is created via a `POST`, the client can specify the `id` of
the Collection. If the `id` is not specified, one is auto-generated by the
server and returned as part of the `Location` response header.

#### (HTTP API) POST `/space/{space_id}/collections/`

Example request (`id` not specified, auto-generated by the server and returned
in the response `Location` header):

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/collections/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"]
}
```
In this example, the `id` in the body is not specified, and will be auto-generated
by the server. The `backend` is not specified, and will be assigned the default
value.


Example response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/21f81693-f4a3-4caa-b81c-b663d6e1e3ae
```

Example request, a valid `id` specified in the body:

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/collections/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "id": "credentials",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "backend": { "id": "default" }
}
```

Example response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/credentials
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=reserved-id=] (409) -- the supplied Collection `id` collides with one of the
  [[[#space-level-reserved-endpoints]]] (for example, `query` or `collections`).
* [=id-conflict=] (409) -- a Collection with the supplied `id` already exists.
  To create or replace a Collection at a client-chosen `id` without conflict,
  use the idempotent [[[#update-or-create-by-id-collection-operation]]] instead.
  Servers MUST perform this existence check only *after* the caller's
  authorization has been verified: an under-authorized caller receives the
  privacy-merged [=not-found=] (404) per [[[#error-handling]]], so that it
  cannot use the `409` to probe a Space for existing Collection ids.
* [=unsupported-backend=] (409) -- the supplied `backend` id is not in that
  space's [[[#space-backends-available]]] list.

### Update (or Create By Id) Collection operation {#update-or-create-by-id-collection-operation}

When creating or modifying a Collection via PUT, the client specifies the `id`
of the Collection. This Collection `id` MUST NOT collide with the list of
[[[#space-level-reserved-endpoints]]].

#### (HTTP API) PUT `/space/{space_id}/{collection_id}`

Example successful "create" request (note the lack of trailing slash):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "backend": { "id": "default" }
}
```

```http
HTTP/1.1 201 Created
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=reserved-id=] (409) -- the supplied Collection `id` collides with one of the
  [[[#space-level-reserved-endpoints]]] (for example, `collections` or
  `linkset`).
* [=encryption-immutable=] (409) -- the update tried to change or clear an
  existing `encryption` marker (the marker is set-once; see
  [[[#collection-data-model]]]).

### Get Collection Description operation {#get-collection-description-operation}

* Returns the Collection description object
* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request must be
    signed by the space's [=controller=], or invoke a delegated capability that
    allows the [=GET=] action

#### (HTTP API) GET `/space/{space_id}/{collection_id}`

Example request (note: no trailing slash):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/linkset",
  "backend": { "id": "default" }
}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Collection does not exist, or the caller has
  missing or insufficient authorization; per [[[#error-handling]]] a Collection
  the caller is not authorized to read is indistinguishable from one that does
  not exist.

### List Collection operation {#list-collection-operation}

* Returns the list of Collection resources
* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request must be
    signed by the space's [=controller=], or invoke a delegated capability that
    allows the [=GET=] action
* MAY be paginated (see [[[#pagination]]]); the example below shows a single
  unpaginated page, and the paginated example that follows shows the `next` link
  and cursor continuation.

#### (HTTP API) GET `/space/{space_id}/{collection_id}/`

Example request (note the trailing slash):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
  "name": "Example JSON Documents Collection",
  "type": ["Collection"],
  "totalItems": 2,
  "items": [
    {
      "id": "321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "contentType": "application/json" 
    },
    {
      "id": "3943c87f-b617-44bc-ba75-8de2b16c3640",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/3943c87f-b617-44bc-ba75-8de2b16c3640",
      "contentType": "application/json" 
    }
  ]
}
```

##### Paginated example

A client requests a bounded page with `limit` (here, two items from a Collection
that holds more):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/?limit=2 HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

The response carries the page of items plus a `next` link, signalling that more
items follow. `totalItems` is omitted here -- the server does not return a total
for this (potentially large) Collection:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
  "name": "Example JSON Documents Collection",
  "type": ["Collection"],
  "items": [
    {
      "id": "321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "contentType": "application/json"
    },
    {
      "id": "3943c87f-b617-44bc-ba75-8de2b16c3640",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/3943c87f-b617-44bc-ba75-8de2b16c3640",
      "contentType": "application/json"
    }
  ],
  "next": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/?limit=2&cursor=eyJhZnRlciI6IjM5NDNjODdmLWI2MTctNDRiYy1iYTc1LThkZTJiMTZjMzY0MCJ9"
}
```

The client follows `next` verbatim to fetch the subsequent page (the `cursor` is
opaque and supplied by the server -- the client does not build it):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/?limit=2&cursor=eyJhZnRlciI6IjM5NDNjODdmLWI2MTctNDRiYy1iYTc1LThkZTJiMTZjMzY0MCJ9 HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

The final page omits `next`, marking the end of the list:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
  "name": "Example JSON Documents Collection",
  "type": ["Collection"],
  "items": [
    {
      "id": "9c7b2e51-0d4a-4c2f-8b3e-1f6a5d8e7c90",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/9c7b2e51-0d4a-4c2f-8b3e-1f6a5d8e7c90",
      "contentType": "application/json"
    }
  ]
}
```

On a Collection that declares an `encryption` marker (see
[[[#encryption-scheme-registry]]]), each item's user-writable metadata is stored
encrypted (see [[[#resource-metadata-data-model]]]). Item summaries for an
encrypted Collection therefore carry only server-visible fields (`id`, `url`,
`contentType`) and omit `name`; a client that needs names decrypts each
Resource's Metadata itself.

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Collection does not exist, or the caller has
  missing or insufficient authorization; per [[[#error-handling]]] a Collection
  the caller is not authorized to read is indistinguishable from one that does
  not exist.

### Delete Collection operation {#delete-collection-operation}

#### (HTTP API) DELETE `/space/{space_id}/{collection_id}`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request
    must either: be signed by the resource's or the space's [=controller=],
    or invoke a delegated capability that allows the `DELETE` action.

* This operation is idempotent

* (Assuming the request carries appropriate authorization) Sending a DELETE
  request to a collection that does not exist (or has already been deleted)
  results in a 204 success response

Example request (note the lack of trailing slash):

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv HTTP/1.1
Host: example.com
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

## Resources and Blobs {#resources-and-blobs}

### Blob Data Model {#blob-data-model}

A unit of data, in transit or at rest.
(As described in the [W3C FileAPI: Blob Interface](https://w3c.github.io/FileAPI/#blob-section)).

Blob properties:

* byte stream
* `size` - length of the byte stream in bytes
  - Note: Although size can be derived from bytes, it's useful to be able to
    have it up front (for the receiving system to decide to reject an upload
    based on quota / exceeding max size, etc).
* `type` (from the IANA mime type registry) - If not specified, defaults to
  `application/octet-stream`

When a Blob is stored as a Resource, its `type` and `size` are reported as
the `contentType` and `size` properties of the Resource's Metadata object
(see [[[#resource-metadata-data-model]]]), and the stored bytes are returned
verbatim on reads (see [[[#content-types-and-representations]]]).

### Resource Data Model {#resource-data-model}

A resource is a named (addressable) Blob stored in a given [=collection=],
with metadata.
The data model is derived from [W3C FileAPI: File
Interface](https://w3c.github.io/FileAPI/#file-section), but with the addition
of a few crucial properties.

In similar storage systems, a resource is called "File", "Object", "Document",
"Row", "Graph", and so on.

Resource properties:

* `id` - A unique identifier for a resource for a given collection. Created by the
  server if not provided. Note: the `{resource_id}` template parameter used in URL
  templates in this spec MUST match that resource's `id` property (as returned by
  the [[[#list-collection-operation]]]).
* `name` - (optional) - Human-readable name for the resource, useful for building
  user interfaces for browsing collection contents.
* `type` (optional) - An array of strings describing the resource. Unlike Space
  and Collection, no specific `type` value is required of a Resource.
* `url` (optional) - A relative URL (provided by the server when listing resources
  via the [[[#list-collection-operation]]])
* `contentType` (optional) - The MIME type of the default representation of the
  resource (provided by the server when listing resources
  via the [[[#list-collection-operation]]])
* Each Resource also has an associated Metadata object -- server-managed
  properties (`contentType`, `size`, optional timestamps) plus user-writable
  ones (`name` and `tags`, nested under `custom`) -- addressable at the
  reserved `/meta` path segment under the Resource URL. See
  [[[#resource-metadata-data-model]]].

### Content Types and Representations {#content-types-and-representations}

A Resource has exactly one current representation: the stored bytes plus
the content type they are stored under. Every successful write (POST or PUT)
replaces that representation entirely -- including replacing a representation
previously stored under a different content type. Servers are not required to
store, convert between, or negotiate multiple representations of a Resource.

**Writing.** A Resource write request MUST carry a `Content-Type` header; a
server MUST reject a write without one with a [=missing-content-type=] (400)
error. The request body is interpreted according to its content type:

* **JSON** -- a content type of `application/json`, or any type with a
  `+json` structured-syntax suffix [[RFC6839]] (for example,
  `application/ld+json`), is stored as a JSON document.
* **Raw binary** (support REQUIRED) -- any other content type is stored as an
  opaque byte stream (see [[[#blob-data-model]]]), verbatim, under the
  supplied content type. Clients with no more specific type SHOULD use
  `application/octet-stream`.
* **Multipart upload** (support OPTIONAL) -- a server MAY additionally accept
  `multipart/form-data` [[RFC7578]] uploads (the HTML form workflow). The
  request MUST contain exactly one file part; that part supplies the stored
  bytes, and its own content type (not `multipart/form-data`) becomes the
  stored content type. Because a write targets a single Resource, there is no
  batch form: a server MUST reject a multipart request with no file part, or
  with more than one, with an [=invalid-request-body=] (400) error.

**Reading.** A `GET` returns the current representation, verbatim, with the
stored content type as the response `Content-Type`. Because Resources are
single-representation, the request's `Accept` header is advisory: a server
MAY ignore it, and MUST NOT reject a Resource read for lack of an acceptable
representation (no `406 Not Acceptable`) -- the stored representation is
always the answer. A `HEAD` request returns the same headers without the
body; the response `Content-Type` and `Content-Length` correspond to the
`contentType` and `size` properties of the Resource's Metadata object (see
[[[#resource-metadata-data-model]]]).

**Description objects.** API documents generated by the server -- Space and
Collection descriptions, listings, quota reports, Metadata objects -- MUST be
available as `application/json`. Linkset documents use
`application/linkset+json` [[RFC9264]], and error responses use
`application/problem+json` [[RFC9457]] (see [[[#error-handling]]]). Other
representations of these documents (negotiated via `Accept`) MAY be offered
in addition to, never instead of, the JSON baseline.

### Create Resource (Add Resource to Collection) Operation {#create-resource-add-resource-to-collection-operation}

#### (HTTP API) POST `/space/{space_id}/{collection_id}/`

Example request (adds a JSON object to the `messages` collection).
Note that since no Resource id was specified, the server auto-generated an id
and returned it as part of the `Location` response header.

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"hi"}
```

Example success response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/6b5be748-5f39-4936-a895-409e393c399c
```

Example request (multipart upload of a photo to the `photos` collection, on
a server that supports the OPTIONAL multipart form; see
[[[#content-types-and-representations]]]). The stored content type is the
file part's own type, `image/png`, not `multipart/form-data`:

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/photos/ HTTP/1.1
Host: example.com
Content-Type: multipart/form-data; boundary=----boundary314
Authorization: ...

------boundary314
Content-Disposition: form-data; name="file"; filename="sunset.png"
Content-Type: image/png

...binary PNG bytes...
------boundary314--
```

Example success response:

```http
HTTP/1.1 201 Created
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/photos/d2887cf0-186f-4e2e-a575-c9ce1b9d8a45
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=missing-content-type=] (400) -- the write request carries no
  `Content-Type` header (see [[[#content-types-and-representations]]]).
* [=invalid-request-body=] (400) -- the body does not match its declared
  content type (for example, a multipart upload with no file part, or with
  more than one).
* [=reserved-id=] (409) -- the supplied Resource `id` collides with one of the
  [[[#collection-level-reserved-endpoints]]] (for example, `query` or
  `linkset`).
* [=id-conflict=] (409) -- a Resource with the supplied `id` already exists.
  To create or replace a Resource at a client-chosen `id` without conflict,
  use the idempotent [[[#update-or-create-by-id-resource-operation]]] instead.
  Servers MUST perform this existence check only *after* the caller's
  authorization has been verified: an under-authorized caller receives the
  privacy-merged [=not-found=] (404) per [[[#error-handling]]], so that it
  cannot use the `409` to probe a Collection for existing Resource ids.
* [=quota-exceeded=] (507) -- the Collection's backend has no storage quota
  remaining (see [[[#quotas]]]).
* [=payload-too-large=] (413) -- the upload exceeds the backend's
  `maxUploadBytes` constraint (see [[[#quotas]]]).

### Read Resource Operation {#read-resource-operation}

A read returns the Resource's single stored representation, with the stored
content type; see [[[#content-types-and-representations]]] for content type
and `Accept` header handling (including the `HEAD` variant).

#### (HTTP API) GET `/space/{space_id}/{collection_id}/{resource_id}`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [=GET=] action

Example request to retrieve a resource:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Accept: application/json
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{"message":"hi"}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Resource does not exist, or the caller has
  missing or insufficient authorization; per [[[#error-handling]]] a resource
  the caller is not authorized to read is indistinguishable from one that does
  not exist.

### Update (or Create By Id) Resource Operation {#update-or-create-by-id-resource-operation}

When creating or modifying a Resource via PUT, the client specifies the `id`
of the Resource. This Resource `id` MUST NOT collide with the list of
[[[#collection-level-reserved-endpoints]]].

#### (HTTP API) PUT `/space/{space_id}/{collection_id}/{resource_id}`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [=PUT=] action
* This operation is idempotent
* Returns a `204` success response

Example request to create a resource via PUT:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"hi"}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Example request to update the created resource via PUT:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"no I've changed my mind"}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Example request (uploading a binary Blob via PUT; the bytes are stored
verbatim under the supplied content type, see
[[[#content-types-and-representations]]]). The `Digest` header binds the
body to the request signature when using the WAS Authorization Profile (see
[[[#request-body-integrity-digest-header]]]):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/photos/sunset.png HTTP/1.1
Host: example.com
Content-Type: image/png
Digest: mh=uEiCPO-qYr-z0GYV5F75-N1l8Rhjv4xIkKZsnbTZeZ7emSA
Authorization: ...

...binary PNG bytes...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=missing-content-type=] (400) -- the write request carries no
  `Content-Type` header (see [[[#content-types-and-representations]]]).
* [=invalid-request-body=] (400) -- the body does not match its declared
  content type (for example, a multipart upload with no file part, or with
  more than one).
* [=reserved-id=] (409) -- the supplied Resource `id` collides with one of the
  [[[#collection-level-reserved-endpoints]]] (for example, `query` or
  `linkset`).
* [=not-found=] (404) -- the enclosing Space or Collection is missing or
  invalid, or the caller has missing or insufficient authorization; per
  [[[#error-handling]]] an under-authorized request is indistinguishable from a
  missing target.
* [=quota-exceeded=] (507) -- the Collection's backend has no storage quota
  remaining (see [[[#quotas]]]).
* [=payload-too-large=] (413) -- the upload exceeds the backend's
  `maxUploadBytes` constraint (see [[[#quotas]]]).
* [=precondition-failed=] (412) -- a conditional write's `If-Match` /
  `If-None-Match` precondition evaluated false, on a backend that advertises
  the `conditional-writes` feature (see [[[#conditional-requests]]]).

This operation accepts the `If-Match` / `If-None-Match` write preconditions
described in [[[#conditional-requests]]] when the target backend advertises the
`conditional-writes` feature.

### Delete Resource Operation {#delete-resource-operation}

#### (HTTP API) DELETE `/space/{space_id}/{collection_id}/{resource_id}`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request
    must either: be signed by the resource's or the space's [=controller=],
    or invoke a delegated capability that allows the
    [=DELETE=] action

* This operation is idempotent

* (Assuming the request carries appropriate authorization) Sending a DELETE
  request to a resource that does not exist (or has already been deleted)
  results in a 204 success response

Example request to delete a resource via DELETE:

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the caller has missing or insufficient authorization.
  Because DELETE is idempotent, an authorized request for an already-absent
  resource returns `204`; a request that lacks sufficient authorization instead
  returns `404`, per [[[#error-handling]]], so that an unauthorized caller
  cannot probe for existence.

### Resource Metadata Data Model {#resource-metadata-data-model}

Each Resource has an associated **Metadata object**, addressable at the
reserved `/meta` path segment under the Resource URL (one of the
[[[#resource-level-reserved-endpoints]]]). The metadata endpoints are
OPTIONAL; a [=server=] that does not implement them SHOULD return an
[=unsupported-operation=] (501) error for requests to `/meta` paths.

The Metadata object is a JSON document containing two groups of
properties:

* **Server-managed properties** appear at the top level of the Metadata
  object. They are maintained by the server as a side effect of Resource
  write operations and are read-only: the
  [[[#update-resource-metadata-operation]]] cannot change them.
* **User-writable properties** are nested under the `custom` property and
  are set by the client via the [[[#update-resource-metadata-operation]]].
  Keeping them in their own object makes the writable surface explicit and
  leaves the top level free for future server-managed properties.

Server-managed properties (`contentType` and `size` are REQUIRED in every
Metadata object; the timestamps are OPTIONAL):

* `contentType` - The MIME type of the stored representation of the Resource,
  recorded from the `Content-Type` header of the request that last wrote the
  Resource's content (`application/octet-stream` if none was provided). The
  same value is reported in [[[#list-collection-operation]]] results and as
  the `Content-Type` header of `GET`/`HEAD` responses for the Resource itself.
* `size` - The length in bytes of the stored representation of the Resource
  (see the `size` property of the [[[#blob-data-model]]]).
* `createdAt` (optional) - The [[RFC3339]] `date-time` at which the Resource
  was created.
* `updatedAt` (optional) - The [[RFC3339]] `date-time` at which the Resource's
  content or user-writable metadata was last modified.

<div class="ednote">
**Versioning is optional and backend-advertised.** This metadata model does not
mandate a stored `etag` / version property. When a backend advertises the
`conditional-writes` feature it exposes a Resource version as an HTTP `ETag`
validator and honors `If-Match` / `If-None-Match` preconditions; see
[[[#conditional-requests]]]. A backend without that feature performs
unconditional last-writer-wins upserts. The broader Transaction (multi-write
atomic) mechanism remains deferred.
</div>

User-writable properties:

* `custom` (optional) - A JSON object holding the user-writable properties
  below. A Metadata object with no user-writable properties set MAY omit
  `custom` or report it as `{}`.
  * `name` (optional) - A human-readable name for the Resource. This is the
    same `name` property defined in the [[[#resource-data-model]]] and
    returned by the [[[#list-collection-operation]]]; updating it here
    updates the name shown in Collection listings.
  * `tags` (optional) - A JSON object of application-defined annotations.
    Keys are strings chosen by the application; values SHOULD be strings, so
    that tags stay cheap to index and filter on. Applications that need
    richer structured metadata SHOULD store it as a Resource in its own
    right rather than in `tags`.

On a Collection that declares an `encryption` marker (see
[[[#encryption-scheme-registry]]]), the user-writable `custom` object is stored
**encrypted**: its value is an envelope of the Collection's declared scheme
(the same envelope profile used for a Resource's content). The server-managed
top-level properties (`contentType`, `size`, timestamps) remain plaintext -- the
server needs them for listings, `GET`/`HEAD` headers, and quotas. The server
validates the `custom` envelope structurally on write (rejecting a plaintext
`custom` with an [=encryption-scheme-mismatch=]) and never decrypts it; a client
holding the keys decrypts `custom` back to `{ name, tags, ... }` after reading.
Because the server cannot read an encrypted `name`, [[[#list-collection-operation]]]
summaries for an encrypted Collection carry no `name` (see there).

A Resource's Metadata object is created and deleted together with the
Resource itself: it comes into existence (with only server-managed
properties) when the Resource is created and is removed when the Resource is
deleted. There is no `DELETE /meta` operation; to clear the user-writable
properties, send a `PUT` with an empty `custom` object (`{ "custom": {} }`)
or an empty body object (`{}`).

### Read Resource Metadata Operation {#read-resource-metadata-operation}

#### (HTTP API) GET `/space/{space_id}/{collection_id}/{resource_id}/meta`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [=GET=] action

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world/meta HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "contentType": "application/json",
  "size": 16,
  "createdAt": "2026-06-10T09:12:00Z",
  "updatedAt": "2026-06-12T13:25:00Z",
  "custom": {
    "name": "Hello World greeting",
    "tags": { "project": "demo", "status": "draft" }
  }
}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Resource does not exist, or the caller has
  missing or insufficient authorization; per [[[#error-handling]]] a Metadata
  object the caller is not authorized to read is indistinguishable from one
  that does not exist.
* [=unsupported-operation=] (501) -- the server does not implement the
  optional metadata endpoints.

### Update Resource Metadata Operation {#update-resource-metadata-operation}

A `PUT` to the `/meta` endpoint is a _full_ replacement of the Metadata
object's `custom` object: the stored `custom` object is replaced by the one
in the request body, so any user-writable property omitted from it is
cleared (and a request body with no `custom` property clears them all).
Server-managed properties are not affected by this operation; a server MUST
ignore any top-level properties other than `custom` present in the request
body (so that a client may read the Metadata object, modify it, and `PUT`
it back without first stripping the server-managed properties).

Unlike the [[[#update-or-create-by-id-resource-operation]]], a `PUT` to
`/meta` does **not** create anything: a Metadata object cannot exist apart
from its Resource, so a `PUT` to the `/meta` path of a nonexistent Resource
returns a [=not-found=] (404) error.

#### (HTTP API) PUT `/space/{space_id}/{collection_id}/{resource_id}/meta`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [=PUT=] action
* This operation is idempotent
* Returns a `204` success response

Example request:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world/meta HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "custom": {
    "name": "Hello World greeting",
    "tags": { "project": "demo", "status": "draft" }
  }
}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Resource does not exist (this operation does not
  create one), or the caller has missing or insufficient authorization,
  per [[[#error-handling]]].
* [=invalid-request-body=] (400) -- the request body is not a JSON object, or
  the `custom` object (or a property within it) does not have the shape
  described in [[[#resource-metadata-data-model]]].
* [=unsupported-operation=] (501) -- the server does not implement the
  optional metadata endpoints.

## Linksets {#linksets}

Linksets (from [[RFC9264]]) serve as the main feature detection and extension
mechanism. They can be discovered, via the `linkset` property, from the following:

* The Space Description object, see [[[#space-data-model]]].
* The Collection Description object, see [[[#collection-data-model]]].

### Space Linkset {#space-linkset}

The space `linkset` resource (one of the [[[#space-level-reserved-endpoints]]]),
located at `/space/{space_id}/linkset` contains a set of links to auxiliary
resources and extension points:

* <dfn data-lt="policy relation" id="policy-rel"><code>policy</code> relation</dfn>
  (`https://wallet.storage/spec#policy`) - A link to the
  `/space/{space_id}/policy` resource, which contains a set of links to access control
  policy documents.
* <dfn id="backends-available">`backends-available`</dfn>
  (`https://wallet.storage/spec#backends-available`) - A link to the
  `/space/{space_id}/backends` "Backends Available" resource.
* <dfn data-lt="quotas relation" id="quotas-rel"><code>quotas</code> relation</dfn>
  (`https://wallet.storage/spec#quotas`) -
  A link to the `/space/{space_id}/quotas` per-backend storage [=quota=] report
  (see [[[#quotas]]]).
* `/space/{space_id}/query` - Reserved for cross-space query operations.

Example space linkset resource request and response:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset HTTP/1.1
Accept: application/linkset+json
Authorization: ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
      "https://wallet.storage/spec#policy": [
        { 
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/policy",
          "type": "application/json"
        }
      ],
      "https://wallet.storage/spec#backends-available": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/backends",
          "type": "application/json"
        }
      ],
      "https://wallet.storage/spec#quotas": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/quotas",
          "type": "application/json"
        }
      ]
    }
  ]
}
```

### Collection Linkset {#collection-linkset}

The collection `linkset` resource (one of the [[[#collection-level-reserved-endpoints]]]),
located at `/space/{space_id}/{collection_id}/linkset` contains a set of links
to auxiliary resources and extension points:

* The [=policy relation=] (`https://wallet.storage/spec#policy`) - A link to the
  `/space/{space_id}/{collection_id}/policy` resource, which contains a set of links
  to access control policy documents.
* <dfn data-lt="backend relation" id="backend-rel"><code>backend</code> relation</dfn>
  (`https://wallet.storage/spec#backend`) - A link to the
  detailed `/space/{space_id}/{collection_id}/backend` "Backend Selected for this
  collection" resource.
* <dfn data-lt="quota relation" id="quota-rel"><code>quota</code> relation</dfn>
  (`https://wallet.storage/spec#quota`) - A
  link to the `/space/{space_id}/{collection_id}/quota` storage [=quota=] report
  for this collection (see [[[#quotas]]]).
* `/space/{space_id}/{collection_id}/query` - (Optional) Reserved for query
  operations within a collection.

Example collection linkset resource request and response:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/linkset HTTP/1.1
Accept: application/linkset+json
Authorization: ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/",
      "https://wallet.storage/spec#policy": [
        { 
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/policy",
          "type": "application/json"
        }
      ],
      "https://wallet.storage/spec#backend": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/backend",
          "type": "application/json"
        }
      ],
      "https://wallet.storage/spec#quota": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/quota",
          "type": "application/json"
        }
      ]
    }
  ]
}
```

## Backends {#backends}

Backends are an **optional** infrastructure concern that is orthogonal to the
hierarchical Spaces Repository > Space > Collection > Resource storage model.

Available backends are registered on the Space level, as a combination of
server-side configuration and client-side "Bring Your Own Storage" registration.

For example, on the server configuration side, a given server might support several
backends -- a file system default backend, an EDV encrypted backend, and a
PostgreSQL database backend. And on the client side, a user might register an
external storage provider by connecting to their Dropbox account.

When a Collection is created, the client can optionally specify the preferred
backend for that Collection. If no preferred backend is specified, one is assigned
by the server (usually the `default` backend).

Backends are an **optional system** that exists to serve advanced use cases that
need fine-grained control over storage configurations.
An implementer or client of a given server can omit the `backend` property when
creating a Collection. By default, if not specified, all Collections are
assigned the `default` backend.

### Backend Data Model {#backend-data-model}

A Backend description object advertises a backend's identity and capabilities,
so that clients can select a suitable backend for each Collection (and so that
storage management UIs can present meaningful choices to users).

Backend description properties:

* `id` (required) - A unique backend identifier (within a given space). The id
  `default` is conventionally used for the server-assigned default backend.
  See [[[#identifiers]]] for additional constraints.
* `name` (optional) - An arbitrary human-readable name for the backend. Does
  not have to be unique.
* `managedBy` (optional) - Who operates the backend. One of:
  - `server` - configured and operated server-side (for example, the server's
    filesystem or database backends). This is the default if unspecified.
  - `external` - a "Bring Your Own Storage" provider registered by the client
    (for example, a connected Dropbox account).
* `storageMode` (optional) - An array of storage modalities the backend
  supports. This specification defines two values: `document` (structured JSON
  resources) and `blob` (opaque binary byte streams). Advertising modalities up
  front prevents mismatches, such as attempting to store a multi-gigabyte
  binary file on a JSON-only document store. If unspecified, clients SHOULD
  assume both modalities are supported.
* `persistence` (optional) - Either `durable` (the engine stores data on
  persistent media, e.g. disk, and it survives a restart) or `volatile` (the
  engine stores data in memory, e.g. a RAM-backed cache tier, and data may not
  survive a restart). Defaults to `durable`. Note that this is a _technical_ property of the storage engine,
  deliberately distinct from any _administrative_ data-retention rules (see the
  editor's note on lifecycle configuration below).
* `features` (optional) - An array of capability tokens advertising optional
  _server affordances_ the backend provides beyond the baseline read/write API,
  so that clients can gate behavior on what the backend actually does. The
  vocabulary is open and additive: a client MUST ignore tokens it does not
  recognize, and MUST treat an absent token (or an absent `features` array) as
  "not supported" rather than assuming a default. Tokens this specification
  defines:
  - `conditional-writes` - the backend enforces `If-Match` / `If-None-Match`
    write preconditions (see [[[#conditional-requests]]]).
  - `blinded-index-query` - the backend serves the blinded-index profile of the
    reserved `query` endpoint.
  - `chunked-streams` - the backend supports chunk addressing for large blobs.

  Each token names something the **server** must actively do. Note that
  client-side encryption is deliberately **not** a backend feature: an encrypted
  document is opaque client-encrypted JSON that any document-capable backend
  stores faithfully with no server cooperation, and whether a given Collection
  is encrypted varies _per Collection_ on the very same backend. Encryption is
  therefore a property of a Collection's data (signaled at the Collection level
  and held in the client's keys), not a capability of the backend. Concretely,
  this signal is the Collection's optional `encryption` marker (see
  [[[#collection-data-model]]]): a non-secret declaration any authorized reader
  discovers from the Collection Description, while the keys stay in the client.

A backend that advertises `conditional-writes` takes on the server-side half of
the client contract in [[[#conditional-requests]]]. It MAY derive the `ETag`
from an internal monotonic version counter (such as an Encrypted Data Vault
document `sequence`); the validator is opaque to clients. It MUST evaluate a
write's `If-Match` / `If-None-Match` precondition atomically with the write
(for example, under a per-Resource lock), so that two concurrent writers cannot
both observe the same prior version and both succeed. An in-process
per-Resource lock satisfies this for a single-instance server only; a
horizontally-scaled deployment needs to coordinate the check-and-write across
instances (e.g. an atomic compare-and-swap on the stored version or a shared
lock). The reference implementation provides single-instance locking only.

<div class="ednote">
The schema of a backend's connection configuration (server-internal
connections vs OAuth-style setup flows for external providers) is not yet
specified.
</div>

### Space Backends Available {#space-backends-available}

The list of [=backends=] registered on a Space is discoverable via the
[=backends-available=] relation in the linkset (see [[[#space-linkset]]]).

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/backends HTTP/1.1
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

[
  {
    "id": "default",
    "name": "Server Filesystem",
    "managedBy": "server",
    "storageMode": ["document", "blob"],
    "persistence": "durable"
  },
  {
    "id": "dropbox",
    "name": "Dropbox",
    "managedBy": "external",
    "storageMode": ["blob"],
    "persistence": "durable"
  },
  {
    "id": "edv",
    "name": "Encrypted Data Vault",
    "managedBy": "server",
    "storageMode": ["document", "blob"],
    "persistence": "durable",
    "features": ["conditional-writes", "blinded-index-query", "chunked-streams"]
  }
]
```

### Collection Backend Selected {#collection-backend-selected}

Each collection has an optional `backend` property that is set during its creation
(see [[[#collection-data-model]]]). If not specified, it is assumed to have the
`id` of `default`. The selected backend is discoverable via the
[=backend relation=] in the collection's linkset (see
[[[#collection-linkset]]]).

<div class="ednote">
**Replicas (planned generalization).** The single `backend` property is
expected to generalize to one or more _replicas_ per Collection, each replica
hosted on a backend, with the current single-backend configuration as the
degenerate one-replica case. (Collections do not "live" on a single backend;
backends are infrastructure endpoints, not a fourth tier of the storage
hierarchy.) That generalization will bring with it:

* Client-evaluated replica roles (`active` vs `available`) -- this
  specification will define the vocabulary for declaring replica topology;
  the logic for _selecting_ the active replica (based on latency, battery,
  network status) belongs to the client SDK.
* A sync-status vocabulary per replica (e.g. `synced`, `syncing`, `stale`);
  the exact state machine, including how "cold boots" of `volatile` backends
  are surfaced, is to be determined.
* A Collection-level `storageMode` declaration, validated against each
  replica's backend at creation time.
* Logical vs physical byte accounting in quota reports: the deduplicated size
  of the user's data vs the total bytes consumed across all replicas. These
  only diverge once replication exists, so the [[[#quotas]]] report currently
  carries plain per-backend usage.
* Concurrency control (`If-Match` ETags) for updates to replica topology,
  building on the per-Resource conditional-writes mechanism (see
  [[[#conditional-requests]]]) generalized to multi-replica writes.
</div>

<div class="ednote">
**Lifecycle configuration (administrative retention).** Retention rules such
as "delete guest data after one hour" (a time-to-live, a grace period, and an
expiry action) are expected to be configurable at the Space or Collection
level. Lifecycle configuration is deliberately distinct from both the
backend's `persistence` property (a technical attribute of the storage engine)
and from access-control [=policy=] (who may act on the data). The property
naming is to be determined, precisely to avoid overloading the term "policy".
</div>

## Quotas {#quotas}

[=Quota=] reporting and enforcement are **optional**, and support is
backend-dependent. The quota API serves two distinct consumers:

* The **hot path**: applications checking whether they have room to write,
  served by a compact per-backend report.
* The **admin path**: storage-manager applications that need a per-collection
  usage breakdown, served by the same endpoint with an `include` query
  parameter.

Quota endpoints follow the same maximum-privacy invariant as the rest of this
specification (see [[[#error-handling]]]): a caller not authorized to read a
Space's quota report MUST receive a [=not-found=] (404) error, never a 403.

### (HTTP API) GET `/space/{space_id}/quotas`

Returns the storage report for a Space, grouped by backend, and is discoverable
via the [=quotas relation=] in the Space's linkset (see [[[#space-linkset]]]).
Each entry in the `backends` array carries:

* `id`, `name`, `managedBy` - identifying properties from the corresponding
  Backend description object (see [[[#backend-data-model]]]).
* `state` - the backend's current condition, one of: `ok`, `near-limit`,
  `over-quota`, or `unreachable`.
* `usageBytes` - total bytes consumed on this backend by this Space.
* `limit` - an object with `capacityBytes` and an `isUnlimited` boolean (when
  `isUnlimited` is `true`, `capacityBytes` MAY be omitted).
* `constraints` (optional) - operational constraints such as
  `maxUploadBytes`, the largest single upload the backend accepts.
* `restrictedActions` - an array of [=actions=] (uppercase HTTP verbs, the
  same vocabulary as the WAS Authorization Profile, see
  [[[#authorization-actions-and-the-root-capability]]]) currently unavailable
  on this backend. For example, a full backend reports `["POST", "PUT"]`
  while still permitting reads and deletes.
* `measuredAt` - when the usage numbers were measured. For `external`
  ("Bring Your Own Storage") backends, the server proxies the provider's
  reported numbers, which may be cached or stale; `measuredAt` lets clients
  judge freshness independently of the report's top-level `respondedAt`.

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/quotas HTTP/1.1
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "respondedAt": "2026-06-12T13:25:00Z",
  "backends": [
    {
      "id": "default",
      "name": "Server Filesystem",
      "managedBy": "server",
      "state": "ok",
      "usageBytes": 524288000,
      "limit": { "capacityBytes": 10737418240, "isUnlimited": false },
      "restrictedActions": [],
      "measuredAt": "2026-06-12T13:25:00Z"
    },
    {
      "id": "dropbox",
      "name": "Dropbox",
      "managedBy": "external",
      "state": "near-limit",
      "usageBytes": 314572800,
      "limit": { "capacityBytes": 367001600, "isUnlimited": false },
      "constraints": { "maxUploadBytes": 157286400 },
      "restrictedActions": [],
      "measuredAt": "2026-06-12T13:24:45Z"
    }
  ]
}
```

### (HTTP API) GET `/space/{space_id}/quotas?include=collections`

With the `include=collections` query parameter, each backend entry
additionally carries a `usageByCollection` array, giving storage-manager
applications a full breakdown in a single request while keeping the hot-path
payload lean:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "respondedAt": "2026-06-12T13:25:00Z",
  "backends": [
    {
      "id": "default",
      "name": "Server Filesystem",
      "managedBy": "server",
      "state": "ok",
      "usageBytes": 524288000,
      "limit": { "capacityBytes": 10737418240, "isUnlimited": false },
      "restrictedActions": [],
      "measuredAt": "2026-06-12T13:25:00Z",
      "usageByCollection": [
        { "id": "credentials", "usageBytes": 419430400 },
        { "id": "inbox", "usageBytes": 104857600 }
      ]
    }
  ]
}
```

### (HTTP API) GET `/space/{space_id}/{collection_id}/quota`

Returns the storage report for a single Collection, scoped to that
Collection's backend (the entry has the same shape as a `backends` array
entry, with `usageBytes` reflecting only this Collection's consumption). It is
discoverable via the [=quota relation=] in the Collection's linkset (see
[[[#collection-linkset]]]).

Not all backends support per-collection accounting. Where unsupported, the
server returns an [=unsupported-operation=] (501) error.

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space or Collection does not exist, or the caller
  is not authorized to read the quota report.
* [=unsupported-operation=] (501) -- the Collection's backend does not support
  per-collection quota accounting.

<section class="appendix">

## Pagination {#pagination}

This appendix is normative. It defines the full pagination profile summarized
in [[[#paginated-list-responses]]].

The list operations -- [[[#list-spaces-operation]]], [[[#list-all-collections-operation]]],
and [[[#list-collection-operation]]] -- return a collection of items in the
common envelope (`url`, `totalItems`, `items`). A Space or Collection may hold
more items than are practical to return in a single response, so servers MAY
paginate these responses, returning one page of items at a time. Pagination
is OPTIONAL: a server that returns every item in a single response (as the
examples in those sections show) is conformant, and a client MUST be prepared
for either behavior.

This profile uses cursor-based (also called "keyset") pagination rather than
numeric offsets. A cursor identifies a position in a stably ordered result set,
so paging stays correct and cheap even as items are concurrently added or
removed, and at any depth into a large collection.

### Ordering

A paginated list operation MUST return items in a stable total order: an
ordering that is deterministic and in which no two items compare equal. The
default order is ascending by item `id` (which is unique within its parent, see
[[[#identifiers]]]). A server MAY offer additional orderings, but every order it
supports for pagination MUST be stable and total -- if the primary sort key is
not unique (for example a timestamp), the server MUST break ties on a unique key
(such as `id`) so that the combined order is total. This stable order is what a
cursor seeks within; an unstable order cannot be paginated correctly.

### Requesting a page

A client requests pagination with two OPTIONAL query parameters:

* `limit` - the maximum number of items the client wants in the page, a
  positive integer. A server applies its own default when `limit` is absent, and
  MAY clamp an oversized `limit` down to a server maximum rather than rejecting
  the request. A server MAY return fewer items than `limit` (including zero) and
  still indicate more pages follow; clients MUST NOT treat a short page as the
  end of the list (see `next`, below).
* `cursor` - an opaque token, obtained from a prior page's `next` value (see
  below), naming the position to continue from. A client MUST treat the cursor as
  opaque: it MUST NOT construct, parse, or modify it, and MUST NOT carry a cursor
  from one collection, ordering, or server to another. A server encodes whatever
  it needs into the cursor to resume the ordered scan (typically the sort-key
  value of the last item already returned).

The first page is requested with no `cursor` (a bare `limit`, or neither
parameter).

### The paginated response

A paginated response carries the usual envelope, plus a `next` member when more
items may follow:

* `next` - a URL the client dereferences (with the same authorization) to
  retrieve the following page. The server bakes the appropriate `cursor` (and
  any `limit`) into this URL, so the client follows it without constructing query
  parameters itself. `next` is present if and only if more items may follow;
  its absence marks the last page. This presence/absence is the authoritative
  end-of-list signal.
* `totalItems` - when present, the total number of items in the entire
  collection, not the number in the current page (the page count is simply the
  length of `items`). Computing an exact total can be expensive at scale, so a
  paginating server MAY omit `totalItems`. A client MUST NOT infer the number of
  pages, or whether more pages exist, from `totalItems`; only `next` is
  authoritative.

A client consumes a multi-page list by following `next` from each response until
a response omits it. A server SHOULD ensure that an item present throughout the
traversal appears exactly once across the pages; an item added or removed
concurrently with the traversal MAY or MAY NOT appear. (Snapshot consistency --
a paginated traversal observing a single point in time -- is permitted but not
required; a server MAY encode a snapshot identifier into the cursor to provide
it.)

A `cursor` that is malformed, or that a server can no longer honor (for example
an expired snapshot), is rejected with an [=invalid-cursor=] (`400`) error. As
with other request-validation failures, a server MUST verify the caller's
authorization for the target before validating the cursor, so the error is
only ever observed by a caller already authorized to list that target; an
under-authorized caller receives the merged [=not-found=] (`404`) instead, per
[[[#error-handling]]].

### Pagination parameters and authorization

A capability authorizes a list [=target=] independent of which page is being
read: a capability that authorizes listing a Space, Collection, or Spaces
Repository authorizes retrieval of *every* page of that list. The `limit` and
`cursor` parameters select a page within an already-authorized target; they do
not narrow, widen, or otherwise change the [=target=] a capability must match
(see [[[#root-capability]]]). A server MUST NOT require a distinct capability per
page.

</section>

<section class="appendix">

## Reserved Path Segment Registry {#reserved-path-segment-registry}

This appendix is normative.

### Space-level reserved endpoints {#space-level-reserved-endpoints}

The following path segments represent reserved API endpoints for [[[#spaces]]]
level operations. Usually, the path segment following the `/space/{space_id}/` prefix
is the id of a Collection. The list of reserved endpoints below means that
collections `id`s MUST NOT collide with the corresponding reserved segments.

| Reserved API Endpoint           | Reserved segment | Purpose                                |
|---------------------------------|------------------|----------------------------------------|
| `/space/{space_id}/policy`      | `policy`         | Access control policy                  |
| `/space/{space_id}/backends`    | `backends`       | Storage backends available             |
| `/space/{space_id}/collections` | `collections`    | List and create collections            |
| `/space/{space_id}/export`      | `export`         | Export (download) space contents       |
| `/space/{space_id}/linkset`     | `linkset`        | Links to auxiliary resources           |
| `/space/{space_id}/query`       | `query`          | Reserved for cross-collection queries  |
| `/space/{space_id}/quotas`      | `quotas`         | Reserved for per-backend quota report  |

If a client attempts to create a collection with an `id` that collides with a
reserved segment list above, the server MUST return a 409 Conflict error.

### Collection-level reserved endpoints {#collection-level-reserved-endpoints}

The following path segments represent reserved API endpoints for [[[#collections]]]
level operations. Usually, the path segment following the
`/space/{space_id}/{collection_id}/` prefix is the id of a Resource. The list of
reserved endpoints below means that resource `id`s MUST NOT collide with the
corresponding reserved segments.

| Reserved API Endpoint                        | Reserved segment | Purpose                             |
|----------------------------------------------|------------------|-------------------------------------|
| `/space/{space_id}/{collection_id}/policy`   | `policy`         | Access control policy               |
| `/space/{space_id}/{collection_id}/backend`  | `backend`        | Storage backend selected            |
| `/space/{space_id}/{collection_id}/linkset`  | `linkset`        | Links to auxiliary resources        |
| `/space/{space_id}/{collection_id}/query`    | `query`          | Query resources within a collection |
| `/space/{space_id}/{collection_id}/quota`    | `quota`          | Storage quota report for collection |

If a client attempts to create a resource with an `id` that collides with a
reserved segment list above, the server MUST return a 409 Conflict error.

### Resource-level reserved endpoints {#resource-level-reserved-endpoints}

The following path segments represent reserved API endpoints for Resource
level operations.

| Reserved API Endpoint                                  | Reserved segment | Purpose                                              |
|--------------------------------------------------------|------------------|------------------------------------------------------|
| `/space/{space_id}/{collection_id}/{resource_id}/meta` | `meta`           | Resource metadata (server-managed and user-writable) |

</section>

<section class="appendix">

## Encryption Scheme Registry {#encryption-scheme-registry}

This appendix is normative.

A Collection's optional `encryption` marker (see [[[#collection-data-model]]])
names a client-side encryption `scheme`. This registry maps each `scheme` token
to the wire format the server can expect for Resources in such a Collection, so
that a [=server=] can hold the [=collection=]'s fail-closed guarantee
*structurally* -- by validating the shape of what is written -- rather than
relying on every client to encrypt correctly. The server never holds key
material and never decrypts; it validates only the non-secret envelope
structure, so this enforcement neither requires nor weakens confidentiality.

| `scheme` | Media type         | Envelope profile                                                                                                                                                                                                                                                                                                                          | Reference                                                      |
|----------|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| `edv`    | `application/json` | An [Encrypted Data Vault](https://identity.foundation/edv-spec/) **Encrypted Document**: a JSON object whose `jwe` member is a JWE in JSON Serialization ([[RFC7516]] §7.2) -- a JSON object carrying at least a `ciphertext` member and either a `recipients` array (general serialization) or a top-level `encrypted_key`/`protected` (flattened serialization). The document MAY also carry EDV bookkeeping members (`id`, `sequence`, `indexed`); these are opaque to the server. | [Encrypted Data Vaults](https://identity.foundation/edv-spec/) |

### Server-side write validation

A [=server=] that recognizes the `scheme` declared by a Collection's
`encryption` marker MUST validate the body of every Resource content write
([=POST=] or [=PUT=]) into that Collection against the scheme's envelope
profile, and MUST reject a non-conforming body -- or a body sent under a
`Content-Type` other than the scheme's registered media type -- with an
[=encryption-scheme-mismatch=] error.

This is the structural fail-closed guarantee: a client that has not encrypted
the body, whether through a bug or an omission, cannot store server-visible
plaintext in an encrypted Collection. The check is purely structural; a server
MUST NOT attempt decryption and MUST NOT inspect the envelope's contents.

The rule applies to a **Resource's stored representation** and, on an encrypted
Collection, to the user-writable `custom` object of its
[[[#resource-metadata-data-model]]] (see the encrypted-Collection note there).
It does not apply to the rest of the server-managed API documents -- Collection
Descriptions, the server-managed top-level Metadata properties, [=policy=]
documents, linksets -- which remain `application/json` regardless of the
Collection's encryption status. For the Metadata object specifically, the
document itself stays a plaintext `application/json` object (so the server keeps
its top-level `contentType`, `size`, and timestamps); only the `custom`
sub-value MUST be a conforming envelope on an encrypted Collection, validated the
same fail-closed way (an [=encryption-scheme-mismatch=] on non-conformance).

As with [=id-conflict=], a server MUST verify the caller's authorization
before validating the envelope, so that [=encryption-scheme-mismatch=] is
observable only to callers already authorized to write at that target; an
under-authorized caller receives the merged [=not-found=] instead, and learns
nothing about the Collection.

### Accepting a marker only when it can be enforced

When a Collection create or update declares an `encryption` marker (see
[[[#update-or-create-by-id-collection-operation]]]), a server SHOULD reject a
`scheme` it does not recognize -- one absent from its supported subset of this
registry -- with an [=unsupported-encryption-scheme=] error, rather than storing
a marker it cannot enforce. This ensures that every marker a server accepts is
one it validates on write: "this Collection is marked encrypted" then
structurally implies "plaintext writes to it are rejected here," closing the gap
that a silently-unenforced marker would reopen.

A server MAY instead choose to store markers for schemes it does not enforce
(treating the marker as fully opaque, per [[[#collection-data-model]]]), but
such a server MUST document that it provides no server-side fail-closed
guarantee for those Collections, leaving the guarantee entirely to clients.

### Validation profile

<div class="informative">

A server MAY implement the `edv` envelope profile with a JSON Schema equivalent
to the following sketch. The outer object MUST carry a `jwe` member; only the
`jwe`'s structural members are checked; their values are opaque ciphertext and
are not interpreted. A plaintext object under `application/json` (one with no
valid `jwe`) fails this profile and is rejected with an
[=encryption-scheme-mismatch=], preserving the fail-closed guarantee.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["jwe"],
  "properties": {
    "jwe": {
      "type": "object",
      "required": ["ciphertext"],
      "properties": {
        "protected": { "type": "string" },
        "iv": { "type": "string" },
        "ciphertext": { "type": "string" },
        "tag": { "type": "string" },
        "encrypted_key": { "type": "string" },
        "recipients": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "header": { "type": "object" },
              "encrypted_key": { "type": "string" }
            }
          }
        }
      },
      "anyOf": [
        { "required": ["recipients"] },
        { "required": ["encrypted_key"] },
        { "required": ["protected"] }
      ]
    }
  }
}
```

</div>

</section>

<section class="appendix">

## Error Type Registry {#error-type-registry}

This appendix is normative.

This specification uses [[RFC9457]] Problem Details for HTTP APIs for error
responses (see [[[#error-handling]]]). The `type` property of a problem
response is a URI identifying the _kind_ of problem. Following [[RFC9457]],
a `type` is reused across operations: a single kind such as
[=invalid-id=] is emitted by Create, Update, and Read operations
across Spaces, Collections, and Resources alike. The per-occurrence specifics
belong in the `errors` array (`detail` and an optional `pointer`), and the
short human-readable summary in `title`.

Each `type` URI is a fragment anchor into this registry. The status codes
listed below are typical; a single kind MAY be returned with more than one
status code depending on the operation.

| `type` URI                                                  | Anchor                                                                      | Typical status | Description                                                                                                                                                                                                                                                                                                                                                                                                      |
|-------------------------------------------------------------|-----------------------------------------------------------------------------|----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `https://wallet.storage/spec#not-found`                     | <dfn id="not-found">not-found</dfn>                                         | 404            | The resource (Space, Collection, or Resource) does not exist, or the caller is not authorized to access it. These two conditions are deliberately indistinguishable -- see the privacy note below.                                                                                                                                                                                                               |
| `https://wallet.storage/spec#invalid-id`                    | <dfn id="invalid-id">invalid-id</dfn>                                       | 400            | A Space, Collection, or Resource `id` is missing or not URL-safe.                                                                                                                                                                                                                                                                                                                                                |
| `https://wallet.storage/spec#reserved-id`                   | <dfn id="reserved-id">reserved-id</dfn>                                     | 409            | A client-supplied `id` collides with a [[[#reserved-path-segment-registry]]] segment.                                                                                                                                                                                                                                                                                                                            |
| `https://wallet.storage/spec#id-conflict`                   | <dfn id="id-conflict">id-conflict</dfn>                                     | 409            | A client-supplied `id` in a `POST` create operation already exists. (Create-or-replace by `id` is done idempotently via `PUT`, which does not conflict.)                                                                                                                                                                                                                                                         |
| `https://wallet.storage/spec#invalid-request-body`          | <dfn id="invalid-request-body">invalid-request-body</dfn>                   | 400            | The request body is missing or invalid (e.g. a required property is absent). Entries in `errors` SHOULD carry a `pointer` to the offending field.                                                                                                                                                                                                                                                                |
| `https://wallet.storage/spec#invalid-cursor`                | <dfn id="invalid-cursor">invalid-cursor</dfn>                               | 400            | A pagination `cursor` query parameter is malformed or can no longer be honored (e.g. an expired snapshot). See [[[#pagination]]].                                                                                                                                                                                                                                                                                |
| `https://wallet.storage/spec#missing-content-type`          | <dfn id="missing-content-type">missing-content-type</dfn>                   | 400            | A required `Content-Type` header is missing.                                                                                                                                                                                                                                                                                                                                                                     |
| `https://wallet.storage/spec#missing-authorization`         | <dfn id="missing-authorization">missing-authorization</dfn>                 | 401            | Required `Authorization` / `Capability-Invocation` headers (or proof of possession) are missing.                                                                                                                                                                                                                                                                                                                 |
| `https://wallet.storage/spec#invalid-authorization-header`  | <dfn id="invalid-authorization-header">invalid-authorization-header</dfn>   | 400            | An `Authorization`, `Capability-Invocation`, or `Digest` header is malformed, unparseable, or failed verification.                                                                                                                                                                                                                                                                                               |
| `https://wallet.storage/spec#controller-mismatch`           | <dfn id="controller-mismatch">controller-mismatch</dfn>                     | 400            | The capability invocation in a Create Space request is not currently authorized by the `controller` supplied in the request body: it is neither signed by that DID nor accompanied by a valid, unexpired delegation chain rooted in it. Servers SHOULD differentiate the cause (chain rooted elsewhere, expired delegation, failed proof) in the `detail` string where they can; see [[[#create-space-errors]]]. |
| `https://wallet.storage/spec#unsupported-backend`           | <dfn id="unsupported-backend">unsupported-backend</dfn>                     | 409            | A requested `backend` id is not in the space's [[[#space-backends-available]]] list.                                                                                                                                                                                                                                                                                                                             |
| `https://wallet.storage/spec#encryption-immutable`          | <dfn id="encryption-immutable">encryption-immutable</dfn>                   | 409            | A Collection update tried to change or clear an existing `encryption` marker. The marker is set-once: declaring it on a Collection that lacks one is allowed, but changing its `scheme` (or clearing it) on a populated Collection would corrupt the stored, client-encrypted Resources. See [[[#collection-data-model]]].                                                                                       |
| `https://wallet.storage/spec#encryption-scheme-mismatch`    | <dfn id="encryption-scheme-mismatch">encryption-scheme-mismatch</dfn>       | 422            | A Resource write into an encrypted Collection had a body (or `Content-Type`) that does not conform to the Collection's declared `encryption` scheme envelope profile. Reachable only by a caller already authorized to write -- see [[[#encryption-scheme-registry]]].                                                                                                                                           |
| `https://wallet.storage/spec#unsupported-encryption-scheme` | <dfn id="unsupported-encryption-scheme">unsupported-encryption-scheme</dfn> | 400            | A Collection create/update declared an `encryption` `scheme` the server does not recognize or support. See [[[#encryption-scheme-registry]]].                                                                                                                                                                                                                                                                    |
| `https://wallet.storage/spec#precondition-failed`           | <dfn id="precondition-failed">precondition-failed</dfn>                     | 412            | A conditional write's `If-Match` / `If-None-Match` precondition evaluated false: the Resource's current version did not match, or a create-if-absent target already exists. Header-driven and distinct from the `409` conflict kinds. See [[[#conditional-requests]]].                                                                                                                                           |
| `https://wallet.storage/spec#quota-exceeded`                | <dfn id="quota-exceeded">quota-exceeded</dfn>                               | 507            | A write was rejected because the target backend's storage quota is exhausted. See [[[#quotas]]].                                                                                                                                                                                                                                                                                                                 |
| `https://wallet.storage/spec#payload-too-large`             | <dfn id="payload-too-large">payload-too-large</dfn>                         | 413            | An upload exceeds the target backend's `maxUploadBytes` constraint (see [[[#quotas]]]). Note that unlike [=quota-exceeded=], this rejection is per-request: smaller uploads may still succeed.                                                                                                                                                                                                                   |
| `https://wallet.storage/spec#unsupported-operation`         | <dfn id="unsupported-operation">unsupported-operation</dfn>                 | 501            | An optional operation that this server or the target backend does not support (for example, a per-collection quota report on a backend without per-collection accounting).                                                                                                                                                                                                                                       |
| `https://wallet.storage/spec#invalid-import`                | <dfn id="invalid-import">invalid-import</dfn>                               | 400            | An uploaded archive is not a valid WAS space export.                                                                                                                                                                                                                                                                                                                                                             |
| `https://wallet.storage/spec#storage-error`                 | <dfn id="storage-error">storage-error</dfn>                                 | 500            | An underlying storage operation failed.                                                                                                                                                                                                                                                                                                                                                                          |
| `https://wallet.storage/spec#internal-error`                | <dfn id="internal-error">internal-error</dfn>                               | 500            | An unexpected server-side fault with no more specific kind.                                                                                                                                                                                                                                                                                                                                                      |

**Privacy: the `not-found` kind is intentionally merged.** Under the principle
of maximum privacy (see [[[#error-handling]]]), an unauthorized client MUST NOT
be able to discover the existence of a Space, Collection, or Resource from an
error response. The [=not-found=] kind therefore covers both
"resource absent" and "invalid authorization"; implementations MUST NOT split
it into distinguishable `type` values, and MUST NOT otherwise let the response
(status code, `title`, or `detail`) reveal whether the resource exists.

This merging applies to "insufficient-authorization" and "absent-authorization"
against an existing target. It does not apply to request- or
credential-validation failures, which describe the request rather than the
target and so MAY use their own precise `type`s -- see [[[#error-handling]]].

**Privacy: `id-conflict` is existence-revealing by nature** -- a `409` confirms
that the supplied id is taken. For Create Collection and Create Resource,
servers MUST therefore verify the caller's authorization before checking for
a conflict, so that only callers already authorized to create at that level can
observe the signal; an under-authorized caller receives the merged
[=not-found=] instead. For Create Space, where any caller permitted to attempt
creation learns the same fact from whether creation succeeds, the disclosure is
inherent -- see the note under [[[#create-space-errors]]].

### Error Response Examples

Canonical example responses for the error kinds defined above, referenced by
the per-operation "Errors" lists throughout this specification. The `title`
strings are illustrative, not normative: for example, the noun in a
[=not-found=] `title` varies with the target (Space, Collection, or Resource),
and a `title` may name the operation that failed. Kinds not shown here
([=missing-content-type=], [=invalid-authorization-header=],
[=invalid-import=], [=storage-error=], [=internal-error=]) follow the same
shape; they will gain examples as their corresponding sections are drafted.

[=not-found=] -- a missing target, or missing or insufficient authorization
(deliberately indistinguishable, see the privacy note above):

```http
HTTP/1.1 404 Not Found
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#not-found",
  "title": "Resource not found or insufficient authorization."
}
```

[=invalid-id=] -- an `id` that is not URL-safe (see [[[#identifiers]]]):

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#invalid-id",
  "title": "Invalid Space id.",
  "errors": [
    {
      "detail": "Space 'id' must be URL-safe.",
      "pointer": "#/id"
    }
  ]
}
```

[=reserved-id=] -- a client-supplied `id` that collides with a segment from the
[[[#reserved-path-segment-registry]]]:

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#reserved-id",
  "title": "Invalid collection id (from reserved list)."
}
```

[=id-conflict=] -- a `POST` create operation supplying an `id` that already
exists:

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#id-conflict",
  "title": "A Collection with this id already exists.",
  "errors": [
    {
      "detail": "Use PUT to create-or-replace a Collection at a chosen id.",
      "pointer": "#/id"
    }
  ]
}
```

[=precondition-failed=] -- a conditional `PUT` whose `If-Match` precondition did
not match the Resource's current version (a concurrent write landed first):

```http
HTTP/1.1 412 Precondition Failed
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#precondition-failed",
  "title": "The resource was modified by another write.",
  "errors": [
    {
      "detail": "Re-read the current resource, re-apply your change, and retry."
    }
  ]
}
```

[=invalid-request-body=] -- a missing or invalid request body (here, a Create
Space request without the required `controller` property):

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#invalid-request-body",
  "title": "Invalid Create Space body.",
  "errors": [
    {
      "detail": "'controller' property is required.",
      "pointer": "#/controller"
    }
  ]
}
```

[=missing-authorization=] -- required authorization (here, a proof of
possession signature on a Create Space request) is missing:

```http
HTTP/1.1 401 Unauthorized
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#missing-authorization",
  "title": "Invalid Create Space request.",
  "errors": [
    {
      "detail": "Valid proof of possession of the 'controller' DID must be provided."
    }
  ]
}
```

[=controller-mismatch=] -- the invocation is not currently authorized by the
DID specified in the `controller` property of a Create Space request body
(the `detail` here is the generic catch-all; servers that can tell SHOULD name
the specific cause instead, e.g. "The delegation chain is rooted in a DID
other than the body's 'controller'." or "The delegated capability has
expired."):

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#controller-mismatch",
  "title": "Invalid Create Space request.",
  "errors": [
    {
      "detail": "The invocation must be authorized by the 'controller' DID in the request body: signed by it, or via a delegation chain rooted in it.",
      "pointer": "#/controller"
    }
  ]
}
```

[=unsupported-backend=] -- a `backend` id that is not part of that space's
[[[#space-backends-available]]] list:

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#unsupported-backend",
  "title": "Unsupported backend id, check the space's 'backends available' list."
}
```

[=encryption-immutable=] -- a Collection update tried to change or clear an
existing `encryption` marker (it is set-once; see [[[#collection-data-model]]]):

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#encryption-immutable",
  "title": "Collection encryption marker is immutable."
}
```

[=quota-exceeded=] -- a write rejected because the target backend's storage
quota is exhausted (see [[[#quotas]]]):

```http
HTTP/1.1 507 Insufficient Storage
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#quota-exceeded",
  "title": "Storage quota exceeded for backend 'default'."
}
```

[=payload-too-large=] -- an upload exceeding the target backend's
`maxUploadBytes` constraint:

```http
HTTP/1.1 413 Content Too Large
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#payload-too-large",
  "title": "Upload exceeds the backend's maximum upload size.",
  "errors": [
    {
      "detail": "Upload size 209715200 exceeds 'maxUploadBytes' of 157286400 for backend 'dropbox'."
    }
  ]
}
```

[=unsupported-operation=] -- an optional operation this server or backend does
not support (here, a per-collection quota report):

```http
HTTP/1.1 501 Not Implemented
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#unsupported-operation",
  "title": "Backend 'default' does not support per-collection quota reports."
}
```

</section>

<section class="appendix informative">

## Goals and Requirements {#goals-and-requirements}

This storage specification is intended to support the following goals and
requirements.

### Local-first and Offline capable storage

Users and apps need to be able to use (provision, set up, and start reading and
writing to) storage spaces without being connected to the internet.

### Storage and sharing of public, permissioned, and private encrypted data

Although the local-first offline functionality is necessary, writing data to
stable internet-accessible URLs for the purposes of sharing them is one of the
primary use cases of this specification.

* A user needs to be able to write data (that is intended to be world-readable)
  to a cloud-accessible URL, and be able to send that URL to intended recipients
  via any out-of-band mechanism such as email, chat, and so on.

* User needs to be able to change or revoke permissions at any point after
  sharing. Note that changed permissions apply only to operations that come after
  the change (this spec is not intended to solve the general problem of DRM).

* The sharing and permission system needs to be primarily based on authorization
  capabilities (zCaps). It also needs support storage-side authorization policies
  (even if only as a way for an authorized client to receive an appropriate zcap)

* The sharing mechanism needs to be flexible and granular. For example, a given
  data resource needs to be: world-readable, or readable by groups or categories,
  or by only those possessing the required authorization capabilities, or by
  no one except the author or controller, etc

* Advanced sharing conditions are also desirable (such as "this share expires
  after X amount of time" or "this is a one-time share and will expire after
  the first successful read request")

### Stored data is opaque to the storage provider

* The spec needs to support (though not require) end-to-end client side
  encryption of the space. For plausible deniability, this might need to include
  all data (even marked as public-readable) is encrypted at rest

### Replication to user-controlled local and cloud servers

* Replication reconciles the first two requirements (data reads and writes must
  be offline-capable, but the data must eventually be able to be shared on the
  web via traditional URLs)

* Replication also provides critical availability and disaster recovery
  functionality

* Replication needs to be multi-primary (to reflect the multi-device and
  multi-client user environment)

* Multi-primary replication requires support for a versioning or conflict
  resolution mechanism

* Data, metadata, and permissions all need to be replicated

* Authorship and data provenance (the ability to tell which user or service
  created or edited a given set of data) must work in this permissioned
  multi-primary-write environment

### Serve as a General Purpose application storage backend

Intended to serve as a storage backend for credential wallets, and any other
client-side (Single Page Applications), server side, desktop, and mobile apps
and services.

### Data Portability

Data written to storage spaces using this specification needs to be portable:

* Authorized agents need to be able to export or backup all the data written,
  including all corresponding metadata and permissions

* The sharing and storage system needs to be able to support web domain
  independent identifiers. That is, a user must be able to share data at a given
  URL, then be able to migrate to a different storage service provider
  (potentially operating on a different web domain than the previous one), and
  the shared permissions to that data must not break after service migration

* While portability (and the not breaking of URLs) is relatively easy to achieve
  via redirect mechanisms (such as HTTP 301 and 302 redirect codes), this
  requires the previous service provider to be alive, available, and cooperative.
  However, this is true only of public-readable URLs, and the moment permissions
  are involved, cross-domain redirects become almost impossible to implement.
  In addition, portability from "dead servers" is also required. That is, if a
  cloud-based service provider disappears (or is otherwise unavailable), but a
  user still has a backup/export available, they should be able to set up another
  storage server (on another web domain or network address), and import/restore
  the data from backup, without shares and permissions breaking. Agents that
  the data was previously shared with must still be able to find the data at
  the new storage server location, and their permissions must still work.

### A Plurality of Data Formats and Protocols

* Spec needs to support the storage of data in any format -- binary files and
  objects, structured documents such as JSON or CBOR, contents of relational
  database tables, graphs, and anything else, all using the same unified
  metadata, sharing, and permission mechanisms.

* Storage-side schema enforcement is available but not required.

* Spec needs to be able to support multiple protocols and APIs, such as HTTP,
  JSON-RPC, DIDComm, local client APIs, and more.

### Permissioned Query and Search functionality

* Where appropriate (such as for unstructured text, structured documents, RDBMSs
  etc), storage needs to be queryable or searchable

* Any query/search mechanism needs to work well with the sharing/permission and
  replication requirements

### Upgradeable and legislation-compliant cryptography

All cryptography has a half-life.

* Any cryptographic operations (such as hashing, signatures, and encryption)
  used in this specification must be able to be obsoleted or upgraded, as
  techniques and algorithms break. To put it another way, the spec cannot
  "hardcode" any given algorithm (although it can recommend current best
  practices)

* Implementations of this spec need to be usable with FIPS-compliant
  cryptographic algorithms

### Anti-Goals

#### Use cases do not include "zero trust" environments

In a "zero trust" storage environment, the sync and replication nodes are
assumed to be untrusted: they hold only ciphertext, enforce no authorization of
their own, and encryption alone serves as the access control mechanism.

This storage specification is intentionally positioned to _not_ be used in such
environments. All encryption has an unpredictable half-life, and some use cases
do not permit relying on encryption only for access control.

What this specification pursues instead is a _combination_ of the two:
encryption protects data at rest and in transit, while minimally trusted
[=servers=] independently enforce the applicable [=policy=] on every request.
A broken cipher alone therefore does not grant an attacker access.

</section>

<section class="appendix informative">

## IANA Considerations {#iana-considerations}

This section will be submitted to the Internet Engineering Steering Group (IESG)
for review, approval, and registration with IANA.
</section>

