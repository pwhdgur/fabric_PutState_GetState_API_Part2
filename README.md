# fabric_PutState_GetState_API_Part2

■ 참조 사이트 : https://medium.com/@kctheservant/putstate-and-getstate-the-api-in-chaincode-dealing-with-the-state-in-the-ledger-part-2-839f89ecbad4

< PutState and GetState: The API in Chaincode Dealing with the State in the Ledger (Part 2) >

- PutState, GetState, GetStateByRange and GetStateByRangeWithPagination 사용해봄.
- the composite key를 사용하여 레코드를보다 효율적으로 찾는 데 어떻게 도움이되는지 살펴 봅니다.
- fabcar 체인 코드를 수정해서 다루어 봅니다.

1. Composite Key
- 현재는 queryCar(), changeCarOwner()와 같은 모든 함수는 carid를 색인으로 사용하여 자동차를 찾습니다.
- 소유자가 "브래드"인 자동차를 쿼리하려는 경우이 키-값 페어 스토리지는 비효율적이여서 Composite Key를 이용하면 효율적입니다.

2. 다양한 Composite Key APIs

- Create Composite Key : func (stub *ChaincodeStub) CreateCompositeKey(objectType string, attributes []string) (string, error)
- Split Composite Key : func (stub *ChaincodeStub) SplitCompositeKey(compositeKey string) (string, []string, error)
- GetState by Partial Composite Key : func (stub *ChaincodeStub) GetStateByPartialCompositeKey(objectType string, attributes []string) (StateQueryIteratorInterface, error)
- GetState by Partial Composite Key with Pagination : func (stub *ChaincodeStub) GetStateByPartialCompositeKeyWithPagination(objectType string, keys []string, pageSize int32, bookmark string) (StateQueryIteratorInterface, *pb.QueryResponseMetadata, error)

3. APIs Directory Setup

$ cd fabric-samples/chaincode
$ cp -r fabcar/go/ testcompositekey/
$ cd testcompositekey
$ mv fabcar.go testcompositekey.go

4. 구현 방법

- marbles02 chaincode 참조
- key: “owner~carid” + Owner + CarID
- value: empty
- Composite key를 처리하는 두 가지 API는 CreateCompositeKey 및 SplitCompositeKey (그림참조 : Composite key)

4.1 Modify the Chaincode

4.1.1 Code example : initLedger() - compositekey

	i := 0
	indexName := "owner~carid"
	for i < len(cars) {
		fmt.Println("i is ", i)
		carAsBytes, _ := json.Marshal(cars[i])
		APIstub.PutState("CAR"+strconv.Itoa(i), carAsBytes)
		ownerCaridIndexKey, err := APIstub.CreateCompositeKey(indexName, []string{cars[i].Owner, "CAR" + strconv.Itoa(i)})
		if err != nil {
			return shim.Error(err.Error())
		}
		value := []byte{0x00}
		APIstub.PutState(ownerCaridIndexKey, value)
		fmt.Println("Added", cars[i])
		i = i + 1
	}

- indexName (line 2) : object type
- the owner (from cars[i].Owner) and carid (CAR+i)
- world state example : owner~caridBradCAR1 => Brad is the owner of CAR1, the final portion CAR1 is the carid

4.1.2 Code example : changeCarOwner() - compositekey

- 나중에 소유자가 여러 자동차를 소유하는 상황을 만들기 위해 changeCarOwner()를 사용 할것입니다.

func (s *SmartContract) changeCarOwner(APIstub shim.ChaincodeStubInterface, args []string) sc.Response {

	if len(args) != 2 {
		return shim.Error("Incorrect number of arguments. Expecting 2")
	}

	indexName := "owner~carid"
	carAsBytes, _ := APIstub.GetState(args[0])
	car := Car{}

	json.Unmarshal(carAsBytes, &car)
	// delete old key
	ownerCaridIndexKey, err := APIstub.CreateCompositeKey(indexName, []string{car.Owner, args[0]})
	if err != nil {
		return shim.Error(err.Error())
	}
	err = APIstub.DelState(ownerCaridIndexKey)
	if err != nil {
		return shim.Error("Failed to delete state:" + err.Error())
	}

	// add new record
	car.Owner = args[1]

	carAsBytes, _ = json.Marshal(car)
	APIstub.PutState(args[0], carAsBytes)

	// add new key
	newOwnerCaridIndexKey, err := APIstub.CreateCompositeKey(indexName, []string{args[1], args[0]})
	if err != nil {
		return shim.Error(err.Error())
	}
	value := []byte{0x00}
	APIstub.PutState(newOwnerCaridIndexKey, value)

	return shim.Success(nil)
}

- carid의 소유자를 기반으로 복합 키를 작성
- 이 복합 키 레코드를 삭제하기 위해 DelState API를 호출
- 새로운 소유자의 새로운 복합 키를 만듭 (Using PutState API)

4.1.3 Code example : createCar() - compositekey

- PutState를 사용하여 새 레코드를 작성할 때 사용할 createCar() 코드도 수정함.

func (s *SmartContract) createCar(APIstub shim.ChaincodeStubInterface, args []string) sc.Response {

	if len(args) != 5 {
		return shim.Error("Incorrect number of arguments. Expecting 5")
	}

	indexName := "owner~carid"

	var car = Car{Make: args[1], Model: args[2], Colour: args[3], Owner: args[4]}

	carAsBytes, _ := json.Marshal(car)
	APIstub.PutState(args[0], carAsBytes)

	ownerCaridIndexKey, err := APIstub.CreateCompositeKey(indexName, []string{args[4], args[0]})
	if err != nil {
		return shim.Error(err.Error())
	}
	value := []byte{0x00}
	APIstub.PutState(ownerCaridIndexKey, value)

	return shim.Success(nil)
}

4.1.4 Code example : queryCarByOwner() - compositekey

- Invoke() 내의 s.queryCarByOwner(APIstub, args) 추가

func (s *SmartContract) Invoke(APIstub shim.ChaincodeStubInterface) sc.Response {

	// Retrieve the requested Smart Contract function and arguments
	function, args := APIstub.GetFunctionAndParameters()
	// Route to the appropriate handler function to interact with the ledger appropriately
	if function == "queryCar" {
		return s.queryCar(APIstub, args)
	} else if function == "initLedger" {
		return s.initLedger(APIstub)
	} else if function == "createCar" {
		return s.createCar(APIstub, args)
	} else if function == "queryAllCars" {
		return s.queryAllCars(APIstub)
	} else if function == "changeCarOwner" {
		return s.changeCarOwner(APIstub, args)
	} else if function == "queryAllCarsWithPagination" {
		return s.queryAllCarsWithPagination(APIstub, args)
	} else if function == "queryCarByOwner" {
		return s.queryCarByOwner(APIstub, args)
	}

	return shim.Error("Invalid Smart Contract function name.")
}

- queryCarByOwner() 기능

func (s *SmartContract) queryCarByOwner(APIstub shim.ChaincodeStubInterface, args []string) sc.Response {

	if len(args) < 1 {
		return shim.Error("Incorrect number of arguments. Expecting 1")
	}

	owner := args[0]
	indexName := "owner~carid"

	resultsIterator, err := APIstub.GetStateByPartialCompositeKey(indexName, []string{owner})
	if err != nil {
		return shim.Error(err.Error())
	}
	defer resultsIterator.Close()

	// buffer is a JSON array containing QueryResults
	var buffer bytes.Buffer
	buffer.WriteString("[")

	bArrayMemberAlreadyWritten := false
	var i int
	for i = 0; resultsIterator.HasNext(); i++ {
		responseRange, err := resultsIterator.Next()
		if err != nil {
			return shim.Error(err.Error())
		}

		_, compositeKeyParts, err := APIstub.SplitCompositeKey(responseRange.Key)
		if err != nil {
			return shim.Error(err.Error())
		}

		returnedCarId := compositeKeyParts[1]

		carAsBytes, _ := APIstub.GetState(returnedCarId)

		// Add a comma before array members, suppress it for the first array member
		if bArrayMemberAlreadyWritten == true {
			buffer.WriteString(",")
		}
		buffer.WriteString("{\"Key\":")
		buffer.WriteString("\"")
		buffer.WriteString(returnedCarId)
		buffer.WriteString("\"")

		buffer.WriteString(", \"Record\":")
		// Record is a JSON object, so we write as-is
		buffer.WriteString(string(carAsBytes))
		buffer.WriteString("}")
		bArrayMemberAlreadyWritten = true
	}
	buffer.WriteString("]")

	fmt.Printf("- queryAllCars:\n%s\n", buffer.String())

	return shim.Success(buffer.Bytes())
}

- Calls GetStateByPartialCompositeKey API with the object type and the partial key, which is owner. (line 10)
- The result : an iterator (resultsIterator)
- Calls SplitCompositeKey to break down the composite key (line 28).
- CompositeKeyParts[0]: owner
- CompositeKeyParts[1]: carid

5. Demonstration

$ cd fabric-samples/basic-network
$ ./teardown.sh
$ ./start.sh
$ docker-compose up -d cli
$ docker exec cli peer chaincode install -n mycc -p github.com/testcompositekey -v 0
$ docker exec cli peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n mycc github.com/testcompositekey -v 0 -c '{"Args": []}' -P "OR('Org1MSP.member')"
$ docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["initLedger"]}'

5.1 queryAllCars()

$ docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["queryAllCars"]}'
- Result : [{"Key":"CAR0", "Record":{"colour":"blue","make":"Toyota","model":"Prius","owner":"Tomoko"}},{"Key":"CAR1", "Record":{"colour":"red","make":"Ford","model":"Mustang","owner":"Brad"}},{"Key":"CAR2", "Record":{"colour":"green","make":"Hyundai","model":"Tucson","owner":"Jin Soo"}},{"Key":"CAR3", "Record":{"colour":"yellow","make":"Volkswagen","model":"Passat","owner":"Max"}},{"Key":"CAR4", "Record":{"colour":"black","make":"Tesla","model":"S","owner":"Adriana"}},{"Key":"CAR5", "Record":{"colour":"purple","make":"Peugeot","model":"205","owner":"Michel"}},{"Key":"CAR6", "Record":{"colour":"white","make":"Chery","model":"S22L","owner":"Aarav"}},{"Key":"CAR7", "Record":{"colour":"violet","make":"Fiat","model":"Punto","owner":"Pari"}},{"Key":"CAR8", "Record":{"colour":"indigo","make":"Tata","model":"Nano","owner":"Valeria"}},{"Key":"CAR9", "Record":{"colour":"brown","make":"Holden","model":"Barina","owner":"Shotaro"}}]

5.2 queryCarByOwner()

$ docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["queryCarByOwner", "Brad"]}'
- Result : [{"Key":"CAR1", "Record":{"colour":"red","make":"Ford","model":"Mustang","owner":"Brad"}}]

- the couchDB : http://localhost:5984/_utils 확인

5.3 changeCarOwner()

$ docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["changeCarOwner", "CAR5", "Brad"]}'
- Result : Chaincode invoke successful. result: status:200

$ docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["queryCarByOwner", "Brad"]}'
- Result : [{"Key":"CAR1", "Record":{"colour":"red","make":"Ford","model":"Mustang","owner":"Brad"}},{"Key":"CAR5", "Record":{"colour":"purple","make":"Peugeot","model":"205","owner":"Brad"}}]

- the couchDB : http://localhost:5984/_utils 확인

