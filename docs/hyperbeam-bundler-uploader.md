# HyperBEAM Bundler Uploader Spec

This document defines how developer tooling should treat a HyperBEAM bundler as
an upload target for permaweb deploys.

## Scope

HyperBEAM uploaders submit signed ANS-104 data items to a node. In the current
LapEE-backed production model, the user pays AO into the node's local ledger;
the operator pays AR to settle the resulting bundle on Arweave L1; the node
withdraws earned AO to its configured beneficiary.

The uploader returns the signed data item ID to the caller, not the enclosing
bundle transaction ID. The item ID is what Arweave manifests and ArNS records
must reference. Once HyperBEAM accepts the item, developer tooling should treat
the upload as accepted; HyperBEAM owns the subsequent bundling, posting, and
settlement flow.

## Discovery

A tool can be configured directly with:

- `uploader-type=hyperbeam`
- `uploader=<node-url>`

Optional discovery can use public Arweave GraphQL:

```graphql
query ($owners: [String!], $tags: [TagFilter!]) {
  transactions(owners: $owners, tags: $tags, first: 10, sort: HEIGHT_DESC) {
    edges {
      node {
        id
        owner {
          address
        }
        tags {
          name
          value
        }
        block {
          height
          timestamp
        }
      }
    }
  }
}
```

with tags `Type=LapEE-Location` and, if known, `Node-URL=<node-url>` or
`Operator=<operator-address>`.

The live known-good node currently advertises:

- `Type=LapEE-Location`
- `Node-URL=https://lapee.hyperzine.xyz`
- `Operator=adx9jZOmbIXmg0dywNpqY9vw8udSpAW13pSPXy5Vy10`
- `AO-Token=0syT13r0s0tgPmIed95bJnuSqaD29HQNN8D3ElLSrsc`
- `AO-Payment-Ledger=2GgZJj_hTvZ5W0H5m7Wu5ZHaywfN9NzWZwWNY2wLggo`

## Preflight

Before uploading, a HyperBEAM-aware client should check:

- `GET <node>/~bundler@1.0/status/body`
  - must return HTTP 200
  - should include `implementation=dev_lapee_bundler`
  - should expose queue/cache pressure fields such as `cache-full`,
    `max-item-bytes`, and `queue-items`
- `GET <node>/~meta@1.0/info/address`
  - must return the operator/node address
- `GET <node>/~location@1.0/node`
  - must return the canonical HyperBEAM location record

Tools should not require raw `POST <node>/graphql`. On the current known-good
image, public discovery works through Arweave GraphQL and `~lapee-location@1.0/query`,
but raw `/graphql` returns 404 and `~query@1.0/graphql` fails to start because
the shipped release is missing the `graphql` OTP application dependency.

## Upload

For each file or manifest:

1. Build an ANS-104 data item locally.
2. Include the normal permaweb tags:
   - `App-Name=Permaweb-Deploy`
   - `Content-Type=<mime-type>`
   - manifest items use `Content-Type=application/x.arweave-manifest+json`
3. Sign the data item locally with an Arweave JWK.
4. Compute the local data item ID from the signed bytes.
5. POST the raw signed bytes:

```http
POST <node>/~bundler@1.0/item?codec-device=ans104@1.0
Content-Type: application/octet-stream
Accept: application/json, text/plain, */*
```

6. If the node returns an ID, it must equal the locally computed data item ID.
   A mismatch is a hard error.
7. Return the local data item ID to the deployment workflow.

Tooling should display the immediate HyperBEAM resolver URL:

```text
<node>/<item-id>
```

For example:

```text
https://lapee.hyperzine.xyz/<item-id>
```

The corresponding `https://arweave.net/<item-id>` URL is expected to become
available only after the node has completed its normal bundling and Arweave
settlement path.

## Payment

HyperBEAM uploaders use AO-paid HyperBEAM conventions:

- Discover payment profile from HyperBEAM metadata where possible.
- Quote by signed item byte length.
- If auto-funding is enabled, transfer AO from the deploy wallet to the node's
  AO deposit address and ingest/wait until local ledger credit is visible.
- If the node returns HTTP 402, show a funding hint containing the deposit
  address, token process, and local ledger route.

Tooling must not assume that the beneficiary address equals the node wallet.
The beneficiary is operator policy and is only relevant to operator settlement
accounting, not to client upload signatures.

## Verification

Successful upload verification has three levels:

1. Local: the node returned success for the signed data item, with no returned
   ID mismatch.
2. Node: `https://lapee.hyperzine.xyz/<item-id>` resolves through the HyperBEAM
   node after acceptance.
3. Public: `https://arweave.net/<item-id>` resolves after the node has settled
   the containing bundle and public gateways have indexed it.

Developer deploy tooling should stop at level 1 for upload success and show the
level 2 URL for immediate reads. Public Arweave verification is useful for
operator monitoring and release tests, but it is not part of the client upload
success boundary.

## Current permaweb-deploy Comparison

Before this update, the HyperBEAM-enabled branch already:

- accepted `--uploader-type hyperbeam`
- signed ANS-104 data items locally
- posted bytes to `/~bundler@1.0/item?codec-device=ans104@1.0`
- supported AO local-ledger funding through `hyperbalance`
- printed useful HTTP 402 funding hints

The branch did not:

- preflight for `dev_lapee_bundler`
- verify returned node IDs against the local signed item ID
- display the immediate HyperBEAM item URL instead of implying that public Arweave
  gateway availability is synchronous
- document that raw HyperBEAM GraphQL is not a deploy-tooling dependency

This implementation makes those checks the `hyperbeam` uploader behavior. There
is no separate `lapee` uploader type.
