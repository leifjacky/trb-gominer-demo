## Tellor(TRB) stratum cpuminer written in golang 

###  2019-03-26 IMPORTANT UPDATE !

#### To make sure nonce can be successfully packed into a utf8 string in solidity, each byte of nonce must be <= 0x7f.

>For example
>
>valid: "000000000000000017582701", "3030313b3e3d303f35393531", "6b436a3d3030323158681a22", "7f7f7f7f3031323300000000"
>
>invalid: "ffffffffffffffff17582701", "8080818b8e8d808f85898581"



For tests only.

### usage

```bash
go get -v github.com/leifjacky/trb-gominer-demo
cd $GOPATH/src/github.com/leifjacky/trb-gominer-demo
go run *.go
```



## Tellor(TRB) stratum protocol

### mining.subscribe

- params: ["agent", null]
- result: [null, "nonce1", nonce2 size]

```json
request:
{
	"id": 1,
	"method": "mining.subscribe",
	"params": ["TRBminer-v1.0.0", null]
}

response:
{
	"id": 1,
	"result": [null, "53252c19", 12],
	"error": null
}
```

nonce1 is first part of the block header nonce (in hex).

We assume the length of nonce is 16 bytes. The miner will pick nonce2 such that len(nonce2) = 16 - len(nonce1) = 16 - 4 = 12 bytes.



### mining.authorize

- params: ["username", "password"]
- result: true

```json
{
	"id": 2,
	"method": "mining.authorize",
	"params": ["0x810a4813068dc8571510071268d90a3d9108a298.worker1", "x"]
}

{"id":2,"result":true,"error":null}
```



### mining.set_difficulty

- params: ["job difficulty"]

```json
{
	"id": null,
	"method": "mining.set_difficulty",
	"params": ["123abc"]		// job difficulty must be integer in hex string, in case of overflow
}
```



### mining.notify

- params: ["jobId", "challenge", "coinbase address", "block difficulty", fresh job]

```json
{
	"id": null,
	"method": "mining.notify",
	"params": ["84557030","93a16ac3eaa54c323dbeccf7c7a9a061daa4d1a3c9b4d8f7fccdffbbbc97ea64","7f97009879cbbbcbd6ca0ced94644d25be4bef15","46720b08e8d81",true]
}
```

>Here, "block difficulty" differs from "job difficulty" which set by "mining.set_difficulty". 
>
>"block difficulty" is integer in hex string, in case of overflow.



### mining.submit

- params: [ "username", "jobId", "nonce2" ]
- result: true / false

```json
{
	"id": 102,
	"method": "mining.submit",
	"params": ["0x810a4813068dc8571510071268d90a3d9108a298.worker1","84557030","000000000000000017582701"]
}

{"id":102,"result":true,"error":null}    // accepted share response
{"id":102,"result":false,"error":[21,"low difficulty",null]}  // rejected share response
```

> nonce2 is the second part of the nonce. 
>
> miner submits only when remainder of solution is equal or smaller than (blockDiff / jobDiff). See below for details.
>
> each byte of nonce must be <= 0x7f





```json
In this example

write to pool: {"id":0,"method":"mining.subscribe","params":["trbminer-v1.0.0",null]}
recv from pool: {"id":0,"result":[null,"53252c19",12],"error":null} // subscribed

write to pool: {"id":0,"method":"mining.authorize","params":["0x810a4813068dc8571510071268d90a3d9108a298.worker1","x"]}
recv from pool: {"id":0,"result":true,"error":null} // authorized

recv from pool: {"id":0,"method":"mining.set_difficulty","params":["123abc"]} // job difficulty set to: 1194684
recv from pool: {"id":0,"method":"mining.notify","params":["84557030","93a16ac3eaa54c323dbeccf7c7a9a061daa4d1a3c9b4d8f7fccdffbbbc97ea64","7f97009879cbbbcbd6ca0ced94644d25be4bef15","46720b08e8d81",true]}

write to pool: {"id":0,"method":"mining.submit","params":["0x810a4813068dc8571510071268d90a3d9108a298.worker1","84557030","000000000000000017582701"]} // share found: "000000000000000017582701"
recv from pool: {"id":0,"result":true,"error":null}	// share accepted

jobDiff = 0x123abc = 1194684
blockDiff = 0x46720b08e8d81 = 1239290005589377
compareRemainder = blockDiff / jobDiff = 1037337074

nonce1 = 0x53252c19
nonce2 = 0x000000000000000017582701
nonce = nonce1 + nonce2 = 0x53252c19000000000000000017582701	// each byte of nonce must be <= 0x7f

challenge = 0x93a16ac3eaa54c323dbeccf7c7a9a061daa4d1a3c9b4d8f7fccdffbbbc97ea64
address = 0x7f97009879cbbbcbd6ca0ced94644d25be4bef15

hashInput = challenge + address + nonce = 0x93a16ac3eaa54c323dbeccf7c7a9a061daa4d1a3c9b4d8f7fccdffbbbc97ea647f97009879cbbbcbd6ca0ced94644d25be4bef1553252c19000000000000000017582701
powHash = hashFn(hashInput) = 0x759002bee9415cdd909447706444bd262971797ceeb86b396b0dce2ce64d7b18

remainder = powHash mod blockDiff = 53175048212017467759802978923387577021972170836971070110060363294245502417688 mod 1239290005589377 = 512689088

When remainder <= compareRemainder (512689088 <= 1037337074), miner should submit the solution. Pool accepts this as a valid share.
```
