## Tellor(TRB) stratum cpuminer written in golang 

For tests only.

### usage

```bash
go get -v github.com/leifjacky/ckb-gominer-demo
cd $GOPATH/src/github.com/leifjacky/ckb-gominer-demo
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
	"result": [null, "cebdeb6e", 12],
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
	"params": ["TRB1qyq2znu0gempdahctxsm49sa9jdzq9vnka7qt9ntff.worker1", "x"]
}

{"id":2,"result":true,"error":null}
```



### mining.set_difficulty

- params: [diff]

```json
{
	"id": null,
	"method": "mining.set_difficulty",
	"params": [""]		// job difficulty must be integer in hex string, in case of overflow
}
```



### mining.notify

- params: ["jobId", "challenge", "coinbase address", "block difficulty", fresh job]

```json
{
	"id": null,
	"method": "mining.notify",
	"params": ["1611", "85666ab512fdf4232063b485ffdb74d032f5c21bcc612b22039af01c805077b2", "7f97009879cbbbcbd6ca0ced94644d25be4bef15", "2f9da8e112a63", true]
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
	"params": ["TRB1qyq2znu0gempdahctxsm49sa9jdzq9vnka7qt9ntff.worker1", "1611", "000000000000000000114026"]
}

{"id":102,"result":true,"error":null}    // accepted share response
{"id":102,"result":false,"error":[21,"low difficulty",null]}  // rejected share response
```

> nonce2 is the second part of the nonce. 
>
> miner submits only when remainder of solution is equal or smaller than (blockDiff / jobDiff). See below for details.





```json
In this example

write to pool: {"id":0,"method":"mining.subscribe","params":["trbminer-v1.0.0",null]}
recv from pool: {"id":0,"result":[null,"cebdeb6e",12],"error":null}

write to pool:  {"id":0,"method":"mining.authorize","params":["0x810a4813068dc8571510071268d90a3d9108a298.worker1","x"]}
recv from pool: {"id":0,"result":true,"error":null}

write to pool: {"id":0,"method":"mining.set_difficulty","params":["abc123"]} // job difficulty set to: 11256099
recv from pool: {"id":null,"method":"mining.notify","params":["12020631","85666ab512fdf4232063b485ffdb74d032f5c21bcc612b22039af01c80500371","7f97009879cbbbcbd6ca0ced94644d25be4bef15","34d1369450ac1",true]} // block difficulty 929170695981761


write to pool: {"id":0,"method":"mining.submit","params":["0x810a4813068dc8571510071268d90a3d9108a298.worker1","12020631","000000000000000000987421"]} // share found: "000000000000000000987421"
recv from pool: {"id":0,"result":true,"error":null} // share accepted


jobDiff = 0xabc123 = 11256099
blockDiff = 0x34d1369450ac1 = 929170695981761
compareRemainder = blockDiff / jobDiff = 82548198

nonce1 = 0xcebdeb6e
nonce2 = 0x000000000000000000987421
nonce = nonce1 + nonce2 = 0xcebdeb6e000000000000000000987421

challenge = 0x85666ab512fdf4232063b485ffdb74d032f5c21bcc612b22039af01c80500371
address = 0x7f97009879cbbbcbd6ca0ced94644d25be4bef15

hashInput = challenge + address + nonce = 0x85666ab512fdf4232063b485ffdb74d032f5c21bcc612b22039af01c805003717f97009879cbbbcbd6ca0ced94644d25be4bef15cebdeb6e000000000000000000987421
powHash = hashFn(hashInput) = 0x087415fe723270647dffe38b0e49b2aff0a9af10dc5e5f472558cd53aa16c15f

remainder = powHash mod blockDiff = 3823608844706510412362965034998918316694949240729139028435068128922200097119 mod 929170695981761 = 76978914

When remainder <= compareRemainder, miner should submit the solution. Pool accepts this as a valid share.
```
