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

For non-multiproof mode and multiproof mode, `data` contains an array with [`QueryResult`](#QueryResult) and the state root (`root`). In case multiproof is disabled, this `data` array contains one element per query element. In case multiproof is enabled, the `data` array contains only one element.

##### `QueryResult`

| Field | Type | Description |
| --- | --- | --- |
| `paths` | Array\<String\> | An array of Merkle paths for the queried items. Each path corresponds to the value at the respective index in the `leaves` array. |
| `leaves` | Array\<String\> | An array of the actual values (leaves) corresponding to the queried paths. Each value is typically represented as a 32-byte hash. |
| `gindices` | Array\<Number\> | An array of generalized indices for each value in the `leaves` array. |
| `proof` | Array\<String\> | An array of proof that are sorted **descending order** by the generalized index. Empty array if `include_proof` is `false`. |


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
  "execution_optimistic": true,
  "finalized": true,
  "data": {
    "root": "0xf1f2f3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1",
    "values": {
      "paths": [".genesis_validators_root", ".validators[100]"],
      "results": [
        "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95",
        {
          "pubkey": "0x8f2a41e5b6c71234abcd5678ef90ff11223344556677889900aabbccddeeff0",
          "withdrawal_credentials": "0x00aa2753bbcc...99",
          "effective_balance": "32000000000",
          "slashed": false,
          "activation_eligibility_epoch": "64",
          "activation_epoch": "64",
          "exit_epoch": "18446744073709551615",
          "withdrawable_epoch": "18446744073709551615"
        }
      ]
    },
    "proofs": {
      "proofs_type": "merkle_proof",
      "results": [
        {
          "paths": [".genesis_validators_root"],
          "leaves": [
            "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95"
          ],
          "gindices": ["65"],
          "proofs": [
            "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
            "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
            "0xcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
            "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"
          ]
        },
        {
          "paths": [".validators[100]"],
          "leaves": [
            "0x12345678deadbeefcafebabef00dabad12345678deadbeefcafebabef00dabad"
          ],
          "gindices": ["8796093023233"],
          "proofs": [
            "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
            "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
            "0xcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
            "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"
          ]
        }
      ]
    }
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
  "data": {
    "root": "0xf1f2f3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1",
    "values": {
      "paths": [".genesis_validators_root", ".validators[100]"],
      "results": [
        "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95",
        {
          "pubkey": "0x8f2a41e5b6c71234abcd5678ef90ff11223344556677889900aabbccddeeff0",
          "withdrawal_credentials": "0x00aa2753bbcc...99",
          "effective_balance": "32000000000",
          "slashed": false,
          "activation_eligibility_epoch": "64",
          "activation_epoch": "64",
          "exit_epoch": "18446744073709551615",
          "withdrawable_epoch": "18446744073709551615"
        }
      ]
    },
    "proofs": [
      {
        "proofs_type": "merkle_multiproof",
        "results": [
          {
            "paths": [".genesis_validators_root", ".validators[100]"],
            "leaves": [
              "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95",
              "0x12345678deadbeefcafebabef00dabad12345678deadbeefcafebabef00dabad"
            ],
            "gindices": ["65", "8796093023233"],
            "proofs": [
              "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
              "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
              "0xcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
              "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
              "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"
            ]
          }
        ]
      }
    ]
  }
}
```