# `postSszQuery` API Specification

> [!Note]
> Implementing this endpoint is a part of [EPF6](https://github.com/eth-protocol-fellows/cohort-six) project by [Jun](https://github.com/syjn99) and [Nando](https://github.com/fernantho). You could check our project [proposal](https://hackmd.io/@junsong/HkQ9XBEHel) to get more context ([motivation](https://hackmd.io/@junsong/HkQ9XBEHel#Motivation), progress, etc.).

> [!CAUTION]
> This document is currently under construction.


## Introduction

Handles multiple queries on `BeaconState` with proof of inclusion.

## Specification

- Endpoint: `/eth/v1/beacon/states/{state_id}/query`
- Method: `POST`

### Parameters

| Name | Type | Description | Possible Values |
|---|---|---|---|
| `state_id` | `string` (path) | State identifier. | Can be one of: "head" (canonical head in node's view), "genesis", "finalized", "justified", \<slot\>, \<hex encoded stateRoot with 0x prefix\> |
    
### Request Body

| Name | Type | Description |
| --- | --- | --- |
| `query` | `object` | A detailed query object. |
| `include_proof` | `boolean` | Whether including a merkle proof or not. Default to false. |
| `multiproof` | `boolean` | Whether using multiproof method for efficiency. Default to false. |

Every request MUST satisfy the JSON schema below, otherwise will return `400 Bad Request`. See [Example sections](#Example) for example requests.

```json
{
  "$schema": "https://json-schema.org/draft-07/schema",
  "title": "Query Object for SSZ QL",
  "description": "Schema for SSZ query object",
  "type": "object",
  "properties": {
    "query": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "path": {
            "type": "string",
            "description": "A merkle path to the item"
          }
        },
        "required": [
          "path"
        ]
      },
      "minItems": 0
    },
    "include_proof": {
      "type": "boolean",
      "description": "Whether including a merkle proof or not. Default to false."
    },
    "multiproof": {
      "type": "boolean",
      "description": "Whether using multiproof method for efficiency. Default to false."
    }
  },
  "required": [
    "query"
  ]
}
```

### Response

| Code | Description | Content |
| --- | --- | --- |
| `200` | Success | Response with [Query result](#Query-Result) |
| `400` | Invalid state ID | N/A |
| `404` | State not found | N/A |
| `500` | Internal server error | N/A |
| `503` | Beacon node is currently syncing and not serving request on that endpoint | N/A |

See [Example sections](#Example) for example responses.

#### Query Result

The content for success query follows typical Beacon API format: it includes `version`, `execution_optimistic`, `finalized`, and `data`. 

For non-multiproof mode, `data` contains an array with [](#QueryResultItem) and the state root (`root`).

For multiproof mode, `data` is an array with [`QueryResultWithMultiProofItem`](#QueryResultWithMultiProofItem).

##### `QueryResultItem`

| Field | Type | Description |
| --- | --- | --- |
| `path` | String | A merkle path to the item. |
| `value` | Any | Result of the query. |
| `proof` | Array\<String\> | An array of proof that are sorted **descending order** by the generalized index. Empty array if `include_proof` is `false`. |


##### `QueryResultWithMultiProofItem`

| Field | Type | Description |
| --- | --- | --- |
| `proof_type` | String | The type of proof. In this case, `merkle_multiproof`, indicating a single proof for multiple values. |
| `paths` | Array\<String\> | An array of Merkle paths for the queried items. Each path corresponds to the value at the respective index in the `leaves` array. |
| `leaves` | Array\<String\> | An array of the actual values (leaves) corresponding to the queried paths. Each value is typically represented as a 32-byte hash. |
| `gindices` | Array\<Number\> | An array of generalized indices for each value in the `leaves` array. |
| `proofs` | Array\<Array\<Any\>\> | An array of proof nodes used in the Merkle multiproof. Each inner array is structured as `[generalized_index, hash_value]`. |
| `state_root` | String | The root of target (anchor) `BeaconState`. |


## Example

### 1. Without Multiproof

Request body:

```json
{
    "query": [
        {
            "path": ".validators[100].withdrawal_credentials"
        },
        {
            "path": ".len(validators)"
        }
    ],
    "include_proof": true,
    "multiproof": false
}
```

Response:

```json
{
    "version": "electra",
    "execution_optimistic": false,
    "finalized": true,
    "data": {
        "result": [
            {
                "path": ".validators[100].withdrawal_credentials",
                "value": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2",
                "proof": [
                    "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2"
                ]
            },
            {
                "path": ".len(validators)",
                "value": 1000,
                "proof": [
                    "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2"
                ]
            }
        ],
        "root": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2"
    }
}
```

### 2. With Multiproof

Request body:

```json
{
    "query": [
        {
            "path": ".genesis_validators_root"
        },
        {
            "path": ".fork.current_version"
        }
    ],
    "include_proof": true,
    "multiproof": true
}
```

Response:

```json
{
  "version": "electra",
  "execution_optimistic": true,
  "finalized": true,
  "data": [
    {
      "proof_type": "merkle_multiproof",
      "paths": [
        ".genesis_validators_root",
        ".fork.current_version"
      ],
      "leaves": [
        "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95",
        "0x0500000000000000000000000000000000000000000000000000000000000000"
      ],
      "gindices": [
        65,
        269
      ],
      "proofs": [
        [
          64,
          "0x5730c65f00000000000000000000000000000000000000000000000000000000"
        ],
        [
          33,
          "0xbe3dc5b7843f6b253970803030a18501814c97ac893ec03560ce4962688f857c"
        ],
        [
          17,
          "0x8d637afd2d258e4d079ca7f00dd4c857a61431af7262ede447b712a25d71e4bb"
        ],
        [
          9,
          "0x09ef197f8757969dce6a9379281f5c5b1ab7aeba924631d6ac5e560e817733c0"
        ],
        [
          5,
          "0x323688e7370b5f72bdadc1cdd2f2f4d9cd7065647a6f54a961def1222684e6d6"
        ],
        [
          3,
          "0xb7edabb0a1e1cb42fe1138558ca73510c890ba5dfc013aca0e2bf9a7c26af0a7"
        ],
        [
          268,
          "0x0400000000000000000000000000000000000000000000000000000000000000"
        ],
        [
          135,
          "0x8e38229b2010e3cde27597c6e4852c4e6cdca82e03574383993cd60250e9ed3a"
        ],
        [
          66,
          "0xe05db90000000000000000000000000000000000000000000000000000000000"
        ],
        [
          32,
          "0x96a9cb37455ee3201aed37c6bd0598f07984571e5f0593c99941cb50af942cb1"
        ]
      ],
      "state_root": "0x2d178ffec45f6576ab4b61446f206c206c837fa3f324ac4d93a3eece8aad6d66"
    }
  ]
}
```