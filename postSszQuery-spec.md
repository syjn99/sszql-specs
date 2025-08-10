# `postStateQuery` & `postBlockQuery` API Specification

> [!Note]
> Implementing these endpoints is a part of [EPF6](https://github.com/eth-protocol-fellows/cohort-six) project by [Jun](https://github.com/syjn99) and [Nando](https://github.com/fernantho). You could check our project [proposal](https://hackmd.io/@junsong/HkQ9XBEHel) to get more context ([motivation](https://hackmd.io/@junsong/HkQ9XBEHel#Motivation), progress, etc.).

> [!CAUTION]
> This document is currently under construction.


## Introduction

Handles multiple queries on `BeaconState` and `BeaconBlock` with proof of inclusion.

## Common objects

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
      "minItems": 1
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
| `200` | Success | Response with the [schema](#Response-Schema) |
| `400` | Invalid ID or malformed request | N/A |
| `404` | Not found | N/A |
| `500` | Internal server error | N/A |
| `503` | Beacon node is currently syncing and not serving request on that endpoint | N/A |

See [Example sections](#Example) for example responses.

#### Response Schema

The content for success query follows typical Beacon API format: it includes `version`, `execution_optimistic`, `finalized`, and `data`. See the table below for the fields in `data`.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `root` | String | Yes | The root value of requested state or block. |
| `values` | `QueryValueResult` | Yes | The query result. See [below](#QueryValueResult) for the schema. |
| `proofs` | Array of `QueryProofResult` | No | The inclusion proofs. Only included when `include_proof` is true. When `multiproof` option is enabled, the array consists of only one `QueryProofResult`. Otherwise, the length is equal to the array of requested path. See [below](#QueryProofResult) for the schema. |

##### `QueryValueResult`

| Field | Type | Description |
| --- | --- | --- |
| `paths` | Array\<String\> | An array of Merkle paths for the queried items. Each path corresponds to the value at the respective index in the `results` array. |
| `results` | Array of any | An array of the actual values in JSON format. |


##### `QueryProofResult`

| Field | Type | Description |
| --- | --- | --- |
| `paths` | Array\<String\> | An array of Merkle paths for the queried items. Each path corresponds to the value at the respective index in the `leaves` array. |
| `leaves` | Array\<String\> | An array of the actual values (leaves) corresponding to the queried paths. Each value is typically represented as a **32-byte hash**. |
| `gindices` | Array\<Number\> | An array of generalized indices for each value in the `leaves` array. |
| `proofs` | Array\<String\> | An array of proof that are sorted **descending order** by the generalized index. |

---

## `postStateQuery` Specification

- Endpoint: `/eth/v1/beacon/states/{state_id}/query`
- Method: `POST`

### Parameters

| Name | Type | Description | Possible Values |
|---|---|---|---|
| `state_id` | `string` (path) | State identifier. | Can be one of: "head" (canonical head in node's view), "genesis", "finalized", "justified", \<slot\>, \<hex encoded stateRoot with 0x prefix\> |
    
---

## `postBlockQuery` Specification

- Endpoint: `/eth/v1/beacon/blocks/{block_id}/query`
- Method: `POST`

### Parameters

| Name | Type | Description | Possible Values |
|---|---|---|---|
| `block_id` | `string` (path) | Block identifier. | Can be one of: "head" (canonical head in node's view), "genesis", "finalized", \<slot\>, \<hex encoded blockRoot with 0x prefix\> |

---

## Example

### 1. Without proof (`include_proof` is false)

Request:

```
POST /eth/v1/beacon/states/finalized/query
```


Request body:

```json
{
    "query": [
        {
            "path": ".genesis_validators_root"
        },
        {
            "path": ".validators[100]"
        }
    ],
    "include_proof": false,
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
            "paths": [
                ".genesis_validators_root",
                ".validators[100]"
            ],
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
        }
    }
}
```


### 2. With proof (non-`multiproof` mode)

Request:

```
POST /eth/v1/beacon/states/finalized/query
```


Request body:

```json
{
    "query": [
        {
            "path": ".genesis_validators_root"
        },
        {
            "path": ".validators[100]"
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
            "paths": [
                ".genesis_validators_root",
                ".validators[100]"
            ],
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
                "paths": [
                    ".genesis_validators_root"
                ],
                "leaves": [
                    "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95"
                ],
                "gindices": [
                    "65"
                ],
                "proofs": [
                    "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
                    "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
                    "0xcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
                    "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
                    "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"
                ]
            },
            {
                "paths": [
                    ".validators[100]"
                ],
                "leaves": [
                    "0x12345678deadbeefcafebabef00dabad12345678deadbeefcafebabef00dabad"
                ],
                "gindices": [
                    "8796093023233"
                ],
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
```

### 2. With proof (`multiproof` mode)

Request:

```
POST /eth/v1/beacon/states/finalized/query
```


Request body:

```json
{
    "query": [
        {
            "path": ".genesis_validators_root"
        },
        {
            "path": ".validators[100]"
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
            "paths": [
                ".genesis_validators_root",
                ".validators[100]"
            ],
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
                "paths": [
                    ".genesis_validators_root",
                    ".validators[100]"
                ],
                "leaves": [
                    "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95",
                    "0x12345678deadbeefcafebabef00dabad12345678deadbeefcafebabef00dabad"
                ],
                "gindices": [
                    "65",
                    "8796093023233"
                ],
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
```