# `postStateQuery` & `postBlockQuery` API Specification

> [!Note]
> Implementing these endpoints is a part of [EPF6](https://github.com/eth-protocol-fellows/cohort-six) project by [Jun](https://github.com/syjn99) and [Nando](https://github.com/fernantho). You could check our project [proposal](https://hackmd.io/@junsong/HkQ9XBEHel) to get more context ([motivation](https://hackmd.io/@junsong/HkQ9XBEHel#Motivation), progress, etc.).


## Introduction

Handles single query on `BeaconState` and `BeaconBlock` with proof of inclusion. Returns as bytes serialized by SSZ.

## Common objects

### Request Body

| Name | Type | Description |
| --- | --- | --- |
| `query` | `string` | A merkle path to the item |
| `include_proof` | `boolean` | Whether including a merkle proof or not. Default to false. |

Every request MUST satisfy the JSON schema below, otherwise will return `400 Bad Request`. See [Example sections](#Example) for example requests.

```json
{
    "$schema": "https://json-schema.org/draft-07/schema",
    "title": "Query Object for SSZ QL",
    "description": "Schema for SSZ query object",
    "type": "object",
    "properties": {
        "query": {
            "type": "string",
            "description": "A merkle path to the item"
        },
        "include_proof": {
            "type": "boolean",
            "description": "Whether including a merkle proof or not. Default to false."
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
| `200` | Success | Response as bytes serialized by SSZ. See [SSZ definition below](#Response-Schema). `Eth-Consensus-Version` is included in header for indicating the active consensus version to which the data belongs. |
| `400` | Invalid ID or malformed request | N/A |
| `404` | Not found | N/A |
| `406` | Accepted media type is not supported. | N/A |
| `500` | Internal server error | N/A |
| `503` | Beacon node is currently syncing and not serving request on that endpoint | N/A |

See [Example sections](#Example) for example responses.

#### Response Schema

We need to define new SSZ containers for the query response. 

First, we define constants below:

| Name | Value | Description |
| --- | --- | --- |
| `SSZ_QL_MAX_TREE_DEPTH` | `uint64(64)` | Maxium depth that SSZ-QL traverses through a merkle tree. Inspired from `VALIDATOR_REGISTRY_LIMIT`(= `2**40`), we need at least `50` to query `.validators[0].activation_epoch`. |
| `SSZ_QL_MAX_RESULT_LENGTH` | `uint64(2**30)` (= 1GB) | Maximum size that the query result can have. |

Here's the definition of necessary container, with descriptions of each fields. Note that `ResponseWithProof` is only used when `include_proof` sets to `true`.


```python
class Proof(Container):
    leaf: Bytes32
    gindex: uint64
    proofs: List[Bytes32, SSZ_QL_MAX_TREE_DEPTH]

class Response(Container):
    root: Bytes32
    result: ByteList[SSZ_QL_MAX_RESULT_LENGTH]
    
class ResponseWithProof(Container):
    root: Bytes32
    result: ByteList[SSZ_QL_MAX_RESULT_LENGTH]
    proof: Proof
```

| Field | Type | Description |
| --- | --- | --- |
| `root` | `Bytes32` | The root value of requested state or block. |
| `result` | `ByteList[SSZ_QL_MAX_RESULT_LENGTH]` | The requested query result. |
| `leaf` | `Bytes32` | An hash value corresponding to the queried path. |
| `gindex` | `uint64` | A generalized index for the queired path. |
| `proofs` | `List[Bytes32, SSZ_QL_MAX_TREE_DEPTH]` | An array of proof that are sorted **descending order** by the generalized index. |

---

## `postStateQuery` Specification

- Endpoint: `/eth/v1/beacon/states/{state_id}/query`
- Method: `POST`
- HTTP Header
    - `accept`: `application/octet-stream`, otherwise it will return `406` error.

### Parameters

| Name | Type | Description | Possible Values |
|---|---|---|---|
| `state_id` | `string` (path) | State identifier. | Can be one of: "head" (canonical head in node's view), "genesis", "finalized", "justified", \<slot\>, \<hex encoded stateRoot with 0x prefix\> |
    
---

## `postBlockQuery` Specification

- Endpoint: `/eth/v1/beacon/blocks/{block_id}/query`
- Method: `POST`
- HTTP Header
    - `accept`: `application/octet-stream`, otherwise it will return `406` error.

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
    "query": ".genesis_validators_root",
    "include_proof": false
}
```

If we decode the response by SSZ:

```json
{
    "root": "0xf1f2f3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1",
    "result": "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95"
}
```

---

Request:

```
POST /eth/v1/beacon/states/finalized/query
```


Request body:

```json
{
    "query": ".validators[100]",
    "include_proof": false
}
```

If we decode the response by SSZ:

```json
{
    "root": "0xf1f2f3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1",
    "result": {
        "pubkey": "0x8f2a41e5b6c71234abcd5678ef90ff11223344556677889900aabbccddeeff0",
        "withdrawal_credentials": "0x00aa2753bbcc...99",
        "effective_balance": "32000000000",
        "slashed": false,
        "activation_eligibility_epoch": "64",
        "activation_epoch": "64",
        "exit_epoch": "18446744073709551615",
        "withdrawable_epoch": "18446744073709551615"
    }
}
```


### 2. With proof

Request:

```
POST /eth/v1/beacon/states/finalized/query
```


Request body:

```json
{
    "query": ".genesis_validators_root",
    "include_proof": true
}
```

If we decode the response by SSZ:

```json
{
    "root": "0xf1f2f3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1",
    "result": "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95",
    "proof": {
        "leaf": "0x4b363db94e286120d76eb905340fdd4e54bfe9f06bf33ff6cf5ad27f511bfe95",
        "gindex": "65",
        "proofs": [
            "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
            "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
            "0xcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
            "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"
        ]
    }
}
```

---

Request:

```
POST /eth/v1/beacon/states/finalized/query
```


Request body:

```json
{
    "query": ".validators[100]",
    "include_proof": true
}
```

If we decode the response by SSZ:

```json
{
    "root": "0xf1f2f3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1",
    "result": {
        "pubkey": "0x8f2a41e5b6c71234abcd5678ef90ff11223344556677889900aabbccddeeff0",
        "withdrawal_credentials": "0x00aa2753bbcc...99",
        "effective_balance": "32000000000",
        "slashed": false,
        "activation_eligibility_epoch": "64",
        "activation_epoch": "64",
        "exit_epoch": "18446744073709551615",
        "withdrawable_epoch": "18446744073709551615"
    },
    "proof": {
        "leaf": "0x12345678deadbeefcafebabef00dabad12345678deadbeefcafebabef00dabad",
        "gindex": "8796093023233",
        "proofs": [
            "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
            "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
            "0xcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
            "0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"
        ]
    }
}
```
