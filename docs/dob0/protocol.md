# DOB/0 Protocol

## Introduction

The first step in the DOB protocol family. The DOB/0 protocol specifies the configuration method and interface format of the decoder on DOB and provides a universal decoder to reduce the development workload of most applications.

## DOB/0 Decoder

### Configuration Method

In the DOB/0 protocol, the issuer should store the decoder configuration information in the `description` field of the Spore Cluster. This piece of configuration information should be a JSON object encoded in UTF-8 format, with the following format:

```json
{
  "description": "This is the description for cluster",
  "dob": {
    "ver": 0,
    "decoder": {
      "type": "code_hash",  // or "type_id"
      "hash": "...",
    },
  "pattern": []  // Any type
  }
}
```

`description`: String type, description information for the Spore cluster.

`dob.ver`: Numeric type, representing the DOB protocol version number, which should always be 0 in DOB/0.

`dob.decoder.type`: String type, specifying the way to find a decoder:

- `code_hash`: Find the cell whose CKB hash value of its data is `dob.decoder.hash`.
- `type_id`: Find the cell whose `type_script.args` is `dob.decoder.hash`. Cell's type script must be [Type ID](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md#type-id).

`dob.decoder.hash`: Prefixed hexadecimal string type. See above.

`dob.pattern`: Any type, determined by the decoder.

Create a DOB/0 cluster via Spore SDK:

```jsx
import { createCluster } from '@spore-sdk/api';
import { bytifyRawString } from '@spore-sdk/helpers/buffer';

const account = ...;
const dob_metadata = {
  description: 'this is the description for cluster',
  dob: {
    ver: 0,
    decoder: {
      type: 'code_hash',
      hash: '...',
    },
  	pattern: [["Age", "number", 1, 1, "range", [0, 100]]],
  }
};

const { txSkeleton, outputIndex } = await createCluster({
  data: {
    name: 'My First DOB Cluster',
    description: bytifyRawString(JSON.strinify(dob_metadata)),
  },
  fromInfos: [account.address],
  toLock: account.lock
});

// sign for txSkeleton
```

### Interface Format

Server will provide two parameters, `DNA` and `Pattern` to the DOB/0 decoder:

`DNA`: byte group type, the data of `DNA` specified in DOB parsed as a hexadecimal string.

`Pattern`: byte group type. If the `Pattern` type specified in Cluster is `string`, this parameter is the data encoded in Utf-8 format. Otherwise, the parameter is the `Pattern` string encoded in JSON format and then in Utf-8 format.

The output of DOB/0 decoders should follow the format:

```json
[
  { "name": "Trait A", "traits": [{ "String": "Trait A Value" }] },
  { "name": "Trait B", "traits": [{ "Number": 100 }] }
]
```

`name` is the unique index of the attribute. If the same `name` appears multiple times, the parser should consider all arrays of its values combined.

When the corresponding `traits` under a `name` has only one element, the parser can consider this element to be a single value directly corresponding to `name`. For situations with a dynamic `traits` array and the length may be one, the decoder should use an inline array to avoid ambiguity.

Each element in `traits` should have one and only one key-value pair representing its type and value. The case of multiple key-value pairs is an undefined behaviour.

## DOB/0 Universal Decoder

### Links

- DOB/0 Decoder Interface https://github.com/sporeprotocol/spore-dob-0
- DOB/0 Universal Decoder https://github.com/sporeprotocol/decoder-template-rust

### DNA Definition

In the DOB/0 protocol, DNA should be stored in the `content` field of the DOB. DNA is in JSON format and can be a string, the first element in an array, or a property in an object:

```json
"df4ffcb5e7a283ea7e6f09a504d0e256"

// or

["df4ffcb5e7a283ea7e6f09a504d0e256"]

// or

{
	"dna": "df4ffcb5e7a283ea7e6f09a504d0e256"
}
```

DNA is an unprefixed hexadecimal string type.

Minting a DOB using the DOB/0 universal decoder via the Spore SDK:

```jsx
import { createSpore } from '@spore-sdk/api';
import { bytifyRawString } from '@spore-sdk/helpers/buffer';

const account = ...;
const dob_cluster_id = ...;
const dob_content = {
  dna: 'df4ffcb5e7a283ea7e6f09a504d0e256'
};

const { txSkeleton, outputIndex } = await createSpore({
  data: {
    content_type: 'dob/0',
    content: bytifyRawString(JSON.strinify(dob_content)),
    cluster_id: dob_cluster_id
  },
  fromInfos: [account.address],
  toLock: account.lock
});

// sign for txSkeleton
```

### Pattern Definition

For the comprehensive consideration of ease of use, readability and reducing CKB occupancy, the universal decoder accepts a JSON array type as a pattern:

```json
[
	/* [
		"name",
		"type",
		"offset",
		"len",
		"pattern",
		"args"
	], */
	[
		"Face",
		"string",
		0,
		1,
		"options",
		["Laugh", "Smile", "Sad", "Angry"]
	],
	[
		"Age",
		"number",
		1,
		1,
		"range",
		[0, 100]
	],
	[
		"BirthMonth",
		"number",
		2,
		1,
		"options",
		[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
	],
	[
		"Lucky Number",
		"number",
		1,
		1, // From same the DNA of the age. Your lucky number and age are related.
		"raw"
	],
	[
		"DNA",
		"string",
		0,
		3,
		"raw"
	],
	[
		"Bio",
		"string",
		3,
		512,
		"utf8"
	]
]
```

The meanings of the elements in each subarray from top to bottom are:

`name`: String type, the name of the attribute.

`type`: String type, the type of attribute.

`offset` and `len`: Numeric types, which together specify a segment in DNA, to derive data based on pattern

`pattern`: String type, optional:

- `options`: Limited options. Randomly select one from the args array as the value of the attribute.
- `range`: The `type` field must be `number`. Randomly selecting a number within [args[0], args[1]).
- `utf8` : The `type` field must be `string`. The DNA segment is parsed according to UTF-8 encoding, ignoring all 0x00 at the end.
- `raw` and `type: number`: Parse the DNA into an unsigned number in little-endian order.
- `raw` and `type: string`: Prefixed hex string data for DNA.

`args`: Optional array type. The function is as above.

### Universal Decoder Deployments

```jsx
{
  // CKB Testnet
  decoder: {
	 type: 'code_hash',
	 hash: '0x32f29aba4b17f3d05bec8cec55d50ef86766fd0bf82fdedaa14269f344d3784a',
	 tx_hash: '0x987cf95d129a2dcc2cdf7bd387c1bd888fa407e3c5a3d511fd80c80dcf6c6b67',
	 out_index: 0,
  }
}
```

## Appendix: A possible way to generate DNA

Creating DNA is obtained through a simple hash operation, and the output byte length is consistent with the length needed by the pattern. To ensure the uniqueness and randomness of DNA, we recommend hashing the combination of the block number/hash and the input outpoint. The specific code is as follows (taking Rust as an example):

```rust
const CKB_HASH_PERSONALIZATION: &[u8] = b"ckb-default-hash";

// By Block Number and Cell Id
let block_number = 12559090u64;
let cell_id = {
  let tx_hash =
		h256!("0xe0cc0c77de31483b27384753ec36a1f413bbbf79535c7605a882d490357de97b");
  let out_index = 0u32;
  let mut hash = Blake2bBuilder::new(8)
		.personal(CKB_HASH_PERSONALIZATION)
		.build();
  hash.update(tx_hash.as_bytes());
  hash.update(&out_index.to_le_bytes());
  let mut cell_id = [0u8; 8];
  hash.finalize(&mut cell_id);
  u64::from_le_bytes(cell_id)
};
let dna = {
  let mut hash = Blake2bBuilder::new(12)
			.personal(CKB_HASH_PERSONALIZATION)
			.build();
  hash.update(&block_number.to_le_bytes());
  hash.update(&cell_id.to_le_bytes());
  let mut dna = [0u8; 12];
  hash.finalize(&mut dna);
  dna.to_vec()
};

// By Block Hash and Outpoint（More random）
let block_hash = h256!("0xb140a915e5fdb9fff472376b82ebcc23b6f0b2508f58ed03fdeed4d563c8eca4");
let out_point = {
  let tx_hash =
			h256!("0xe0cc0c77de31483b27384753ec36a1f413bbbf79535c7605a882d490357de97b");
  let out_index = 0u32;
  OutPoint::new_builder()
			.tx_hash(tx_hash.pack())
			.index(out_index.pack())
			.build()
};
let dna = {
  let mut hash = Blake2bBuilder::new(12)
  .personal(CKB_HASH_PERSONALIZATION)
  .build();
  hash.update(&block_hash);
  hash.update(&out_point.as_bytes());
  let mut dna = [0u8; 12];
  hash.finalize(&mut dna);
  dna.to_vec()
};
```
