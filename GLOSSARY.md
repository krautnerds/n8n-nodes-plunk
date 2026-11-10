# n8n-nodes-plunk

An n8n community node package that connects n8n workflows to the Plunk email platform.

## Language

**Plunk instance**:
One running Plunk deployment the node talks to, identified by its base URL; either Plunk Cloud or a self-hosted instance.
_Avoid_: server, account, workspace

**Plunk Cloud**:
The Plunk instance hosted by the Plunk team.
_Avoid_: SaaS Plunk, useplunk.com API

**Self-hosted instance**:
A Plunk instance run by the user from Plunk's open-source code.
_Avoid_: on-prem, local Plunk (except for the test environment)

**Action node**:
The `Plunk` n8n node that performs operations against a Plunk instance.
_Avoid_: regular node, main node

**Trigger node**:
The `Plunk Trigger` n8n node that starts a workflow when something happens in a Plunk instance.
_Avoid_: webhook node, listener

**Resource / Operation**:
The n8n pairing that structures the Action node: a Resource is a Plunk concept (e.g. contact), an Operation is a verb on it (e.g. create).
_Avoid_: endpoint, action, method

**Verified community node**:
A community node approved by n8n so it is installable from the n8n node panel, including on n8n Cloud.
_Avoid_: official node, certified node
