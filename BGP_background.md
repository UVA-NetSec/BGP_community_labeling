# BGP Community Semantic Inference

## 1. Purpose of This Guide

The goal is to understand enough about BGP communities, our semantic schema, and our interpretation rules to review outputs produced by our semantic extraction/inference pipeline.

Your initial task will mainly involve **judging whether semantic information extracted from BGP community documentation is correct**.

For each example, you may see information such as:

```text
BGP community: 1299:30700
Description: Customer routes received in Frankfurt
Defining ASN: 1299
```

along with structured semantic fields produced by the system.

Your job is not simply to determine 
> **Is every populated semantic field supported by the description?**

You should also determine whether the system:

- extracted all relevant information,
- missed information that should have been extracted,
- added unsupported information (Hallucination),
- used the correct schema field,
- used the correct canonical value, and
- assigned the correct `evidence_type`.

The most important principle throughout this guide is:

> **Every accepted semantic value must have identifiable/understandable evidence or an explicitly authorized normalization rule.**

---
# 2. Background: BGP 

## 2.1 Autonomous Systems and ASNs

The Internet is made up of many independently operated networks.

These include:

- Internet Service Providers (ISPs),
- cloud providers,
- content providers,
- enterprises, and
- other organizations.

In Internet routing, these networks are organized into **Autonomous Systems (ASes)**.

Each Autonomous System is identified by an **Autonomous System Number (ASN)**. Examples include: ```AS1299, AS13335```

For this project, the important idea is:

> **An ASN identifies a network participating in Internet routing.**

You will frequently encounter ASNs in BGP community documentation.

For example: ```Received from AS3491```. Here, `3491` identifies an Autonomous System associated with the route.

---

## 2.2 What Is BGP?

**BGP (Border Gateway Protocol)** is the protocol Autonomous Systems use to exchange routing information.

Consider:

```text
AS1 -------- AS2 -------- AS3
```

Suppose AS1 originates an IP prefix:

```text
192.0.2.0/24
```

AS1 can advertise information about that prefix to AS2, and AS2 can advertise the route onward to AS3.

Conceptually:

```text
AS1  →  AS2  →  AS3
```

A BGP route contains an IP prefix together with several attributes.

A simplified example might look like:

```text
Prefix:       192.0.2.0/24
AS Path:      AS2 AS1
Communities:  64500:100
```

Our project primarily focuses on the last attribute:

> **BGP Communities**

---

## 2.3. What Is a BGP Community?

A BGP community is a **tag attached to a BGP route**. For example: ```1299:30700```

Network operators can attach communities to routes to encode information or request routing actions.

A community might describe information such as:

```text
Route learned from a customer
Route learned from a peer
Route learned in Frankfurt
Route received from AS3491
Route received through a route server
```

Other communities may request actions such as:

```text
Do not advertise this route
Prepend the AS path
Blackhole this route
Lower local preference
```

The numeric community value alone generally does not tell an outside observer its complete meaning. Operators therefore often publish documentation describing how their communities are used.This documentation provides an important source of known semantic information for our project.

---

## 2.4. Informational vs. Action Communities

At a coarse level, communities can be categorized as: ```Information/Action```

Our current work focuses on **informational communities**.

### 2.4.1 Informational Communities

Informational communities describe some property of a route.

Examples include:

```text
Received from customer
Received from peer
Learned in Frankfurt
Received from AS3491
Received through a route server
RPKI invalid routes
```

These descriptions may provide information about:

- route source,
- neighboring AS,
- AS relationship,
- route server,
- Internet Exchange Point (IXP),
- geographic location, or
- route validation status.

---

### 2.4.2 Action Communities

Action communities request or control routing behavior.

Examples include:

```text
Do not advertise to peers
Prepend three times
Blackhole this route
Lower local preference
Suppress advertisement
```

These are **not** the semantic information we are trying to extract in the current task.

For example:

```text
Received from peer
```

describes a property of the route.

It is informational.

By contrast:

```text
Do not advertise to peers
```

is an instruction controlling what should happen to the route. It is an action.


## 2.5. Why We Focus on Informational Communities

The broader project focuses on informational communities rather than action communities.

Action communities are commonly used to request routing operations and may be removed after the requested action is performed.

Informational communities, in contrast, are useful for understanding properties associated with observed routes.

Our goal is to recover **fine-grained semantic labels** associated with these informational communities.

---


# 3. Semantic Labels

A **semantic label** is a canonical identifier representing some meaning associated with a community.

Examples include:

```text
Frankfurt
DE-CIX
Customer
Peer
```

A single description may contain **multiple semantic concepts**.

For example:

```text
Customer route learned in Frankfurt
```

contains information related to:

```text
Route Source
Neighbor Relationship
Location
```

Our goal is therefore not simply to classify the entire sentence.

We want to represent its individual semantic components using a structured schema.

---

# 4. From Documentation to Semantic Labels

Documented community semantics provide an important starting point.

Conceptually:

```text
Operator Documentation
        │
        ▼
Extract Semantic Labels
        │
        ▼
Manual Verification
        │
        ▼
Fill Missing Labels
        │
        ▼
Map Labels to Taxonomy
```

For example:

```text
Customer route learned in Frankfurt
```

may contain semantic labels corresponding to:

```text
Import
Customer
Frankfurt
```

which belong to categories such as:

```text
Import     → Route Source
Customer   → Neighbor Relationship
Frankfurt  → Location
```

Human review is important because an automated system can:

- correctly extract information,
- miss supported information,
- assign information to the wrong field,
- apply an incorrect normalization,
- misunderstand ambiguous wording, or
- introduce unsupported information.

---

# 5. BGP Community Semantic Model

At a high level, our semantic model is:

```text
Community
│
├── Type
│
├── Namespace
│   ├── Defining AS
│   ├── Global Administrator
│   ├── Local Administrator
│   └── Value
│
└── Semantics
    ├── Classification
    ├── Route Source
    ├── Neighbor
    ├── Route Server
    ├── Location
    └── Validation Status
```

The following sections explain these fields.

---

# 6. Community Type and Namespace

The schema supports the following community types:

```text
regular
extended
large
```

The namespace contains:

```text
defining_as
global_administrator
local_administrator
value
```

For initial semantic-review work, you do not need to understand all protocol-level differences among regular, extended, and large communities.

The important conceptual field is:

```text
defining_as
```

This identifies the ASN that defines the community.

Several semantic interpretations, particularly neighbor relationships, are expressed from the perspective of this defining AS.

---

# 7. Classification

The schema uses:

```text
classification = information | action
```

For example:

```text
Routes learned from peers
```

is informational.

Therefore:

```text
classification = information
```

By contrast:

```text
Do not announce to peers
```

describes an action.

Our current extraction task focuses on informational semantics and should not extract action semantics such as:

```text
blackhole
prepend
no-export
suppress
deprefer
local-preference changes
do-not-announce
```

---

# 8. Route Source

The schema represents route source as: ```route_source = origin | import```

This field describes the direction of the route relative to the **defining AS**.

---

## 8.1 Import

```text
route_source = import
```

means that the route was received or learned from another network.

Descriptions such as:

```text
learned from
received from
imported from
accepted from
routes from
prefix from
```

normally indicate:

```text
route_source = import
```

For example:

```text
Routes learned from peers
```

supports:

```text
route_source = import
```

---

## 8.2 Origin

```text
route_source = origin
```

means that the description indicates that the **defining AS itself** originates or generates the route or prefix.

Examples include descriptions such as:

```text
AS1764 originated v6 routes
Internal prefixes originated in Ufa
Own Networks in Nizhny Novgorod
```

when the wording clearly indicates prefixes originated by the defining AS.

---

# 8.3. Be Careful With the Word "Origin"

Do **not** determine `route_source` simply because the word:

```text
origin
```

or:

```text
originated
```

appears.

You must determine **whose route is being described**.

For example:

```text
origin: upstream
```

describes an external source.

Therefore:

```text
route_source = import
```

Similarly:

```text
prefixes originated from TTK-NN peer
```

indicates that the route came from an external peer.

Therefore:

```text
route_source = import
```

By contrast:

```text
AS1764 originated v6 routes
```

can indicate:

```text
route_source = origin
```

because the defining AS itself is identified as originating the routes.

The important question is:

> **Is the defining AS originating the route, or is the route coming from another network?**

---

# 9. Relationship Alone Does Not Establish Route Source

This is an important review rule.

A description containing:

```text
customer
```

does **not automatically mean**:

```text
route_source = import
```

Similarly:

```text
peer
provider
transit
upstream
```

alone do not establish route direction.

For example: description ```customer```

supports a relationship interpretation, but: ```route_source = null``` because the description does not state that a route was received from the customer.

Similarly:

```text
Relationship type: Transit
```

does not by itself establish route source.

And:

```text
private peer
```

does not by itself establish route source.

However:

```text
Routes learned from customers
```

does establish that the routes came from customers.

Therefore:

```text
route_source = import
```

The difference is the explicit indication that the route was **learned from** another network.

---
# 10. Neighbor Information

The schema represents neighbor information as:

```text
neighbor:
    asn
    relation
```

These fields answer two different questions:

```text
neighbor.asn
    → Which neighboring AS?

neighbor.relation
    → What is its relationship with the defining AS?
```

---

# 10.1. Neighbor Relationships

The schema represents relationships as:

```text
p2c
c2p
p2p
```

These are interpreted from the perspective of the **defining AS**.

The project uses the following mappings:

| Description | Schema Value | Meaning from Defining AS Perspective |
|---|---|---|
| `customer` / `client` | `p2c` | Provider-to-Customer |
| `provider` / `transit` / `upstream` | `c2p` | Customer-to-Provider |
| `peer` / `peering` | `p2p` | Peer-to-Peer |

For example:

```text
Received from customer
```

supports:

```text
neighbor.relation = p2c
```

while:

```text
Received from upstream
```

supports:

```text
neighbor.relation = c2p
```

and:

```text
Received from peer
```

supports:

```text
neighbor.relation = p2p
```

These mappings are explicitly authorized normalizations.

Therefore, applying them does **not** make the output inferred.

---

# 10.2. Neighbor ASN

Do **not** treat every ASN appearing in a description as:

```text
neighbor.asn
```

An ASN should be extracted as `neighbor.asn` when the description identifies that ASN as:

- the network from which the route was directly received or learned,
- a customer or client,
- a peer,
- a provider,
- an upstream,
- a transit network, or
- a neighbor.

For example:

```text
received from AS3491
```

supports:

```text
neighbor.asn = 3491
```

Similarly:

```text
peer AS3491
```

supports:

```text
neighbor.asn = 3491
```

However:

```text
origin AS3491
```

does **not automatically** establish:

```text
neighbor.asn = 3491
```

And:

```text
AS path contains AS3491
```

does **not** establish:

```text
neighbor.asn = 3491
```

The reviewer must understand the **role of the ASN in the description**, not merely detect that an ASN is present.

---

# 11. Internet Exchange Points (IXPs)

An **Internet Exchange Point (IXP)** is infrastructure where multiple networks can interconnect and exchange traffic.

Examples include:

```text
AMS-IX
LINX
DE-CIX
```

A description may explicitly identify an IXP.

For example:

```text
peer at AMS-IX
```

contains IXP information.

However, an IXP name alone does **not** establish that a route server was involved.

This distinction is important.

---

# 12. Route Servers

The schema represents route-server information as:

```text
route_server:
    asn
    ixp
```

Populate route-server fields only when the description explicitly indicates route-server semantics.

Examples of such wording include:

```text
route server
RS
```

or an explicit statement identifying an ASN as a route-server ASN.

For example:

```text
received from AMS-IX route server
```

supports:

```text
route_server.ixp = AMS-IX
```

Similarly:

```text
received at DECIX-MARSEILLE RS
```

supports:

```text
route_server.ixp = DECIX-MARSEILLE
```

However:

```text
peer at AMS-IX
```

does **not** establish route-server semantics.

Therefore, do not populate:

```text
route_server
```

merely because an IXP appears in the description.

---

# 13. Location

The schema represents geographic information using:

```text
location:
    city
    country
    region
```

A description may contain geographic information such as:

```text
Frankfurt
Germany
London
Europe
NYC
```

When a city, country, metro, continent, region, PoP, facility, or other geography is explicitly mentioned, relevant location information can be extracted.
---

# 14. Geographic Normalization

A geographic value does not always need to appear literally in the description to count as `extracted`.Clear and standard geographic enrichment is permitted.

For example: ```Germany``` can be normalized to:

```text
country = Germany
region = Europe
```

Similarly:

```text
Madrid, Spain
```

can support:

```text
city = Madrid
country = Spain
region = Europe
```
---
# 15. Ambiguous Geographic Interpretation

Geographic normalization becomes **inference** when the location is ambiguous and resolving it requires choosing among multiple plausible interpretations.

For example:

```text
San Jose
```

may refer to different geographic locations.

Choosing:

```text
country = USA
```

without sufficient disambiguating evidence requires an interpretation.

Similarly:

```text
Luxembourg
```

may refer to the country or Luxembourg City.

If the description does not distinguish them, resolving the ambiguity requires inference.

The key distinction is:

```text
Clear + standard + unambiguous normalization
        ↓
extracted

Ambiguous interpretation requiring contextual choice
        ↓
inferred
```

---

# 16. Validation Status

The schema includes:

```text
validation_status:
    rpki
    aspa
    irr
```

The allowed values are:

```text
RPKI:
    valid
    invalid
    invalid-morespecific
    unknown

ASPA:
    valid
    invalid
    unknown

IRR:
    valid
    invalid
    unknown
```

Only populate these fields when the corresponding validation mechanism is supported by the description.

For example:

```text
RPKI invalid routes
```

supports:

```text
validation_status.rpki = invalid
```

RPKI status should only be populated when the description mentions:

```text
RPKI
ROA
origin validation 
(or similar)
```

ASPA status should only be populated when ASPA is mentioned.

IRR status should only be populated when the description refers to concepts such as:

```text
IRR
route object
route registry
registry validation
```

Do not invent a validation state when the corresponding validation mechanism is not present.

---
