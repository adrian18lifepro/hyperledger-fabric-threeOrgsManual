# Customise the Fabric test network
## The original network
Hyperledger-fabric tutorial provides a sample Fabric network which could deployed by Docker Compose. 

`./network.sh up` would create a Fabric network that consists of two peer nodes, one ordering node.

`./network.sh up -ca` can also bring up the network with Certificate Authorities.

## Customisation
Here I want to change some configuration files to bring three peer nodes with CA at once, join them to a channel, deploy chaincode and test client application.

This is the port set up.

Name | CA-PORT | Port | Port-channel
---|---|---|---| 
Org1|7054|7051|7052
Org2|8054|9051|9052
Org3|6054|6051|6052
Orderer|9054|7050


Files changed include: (if currently inside test-network folder)
### (1) ./configtx/configtx.yaml
Section: Organizations
 10 #
 11 #   - This section defines the different organizational identities which will
 12 #   be referenced later in the configuration.

add Org3 information in this section.

```
116     - &Org3
117         # DefaultOrg defines the organization which is used in the sampleconfig
118         # of the fabric.git development environment
119         Name: Org3MSP
120 
121         # ID to load the MSP definition as
122         ID: Org3MSP
123 
124         MSPDir: ../organizations/peerOrganizations/org3.example.com/msp
125 
126         # Policies defines the set of policies at this level of the config tree
127         # For organization policies, their canonical path is usually
128         #   /Channel/<Application|Orderer>/<OrgName>/<PolicyName>
129         Policies:
130             Readers:
131                 Type: Signature
132                 Rule: "OR('Org3MSP.admin', 'Org3MSP.peer', 'Org3MSP.client')"
133             Writers:
134                 Type: Signature
135                 Rule: "OR('Org3MSP.admin', 'Org3MSP.client')"
136             Admins:
137                 Type: Signature
138                 Rule: "OR('Org3MSP.admin')"
139             Endorsement:
140                 Type: Signature
141                 Rule: "OR('Org3MSP.peer')"
142 
143         # leave this flag set to true.
144         AnchorPeers:
145             # AnchorPeers defines the location of peers which can be used
146             # for cross org gossip communication.  Note, this value is only
147             # encoded in the genesis block in the Application section context
148             - Host: peer0.org3.example.com
149               Port: 6051
```

346 #   Profile
347 #
348 #   - Different configuration profiles may be encoded here to be specified
349 #   as parameters to the configtxgen tool

change configuration profiles (TwoOrgsOrdererGenesis ->  ThreeOrgsOrdererGenesis; TwoOrgsChannel -> ThreeOrgsChannel)

```
378     ThreeOrgsOrdererGenesis:
379         <<: *ChannelDefaults
380         Orderer:
381             <<: *OrdererDefaults
382             Organizations:
383                 - *OrdererOrg
384             Capabilities:
385                 <<: *OrdererCapabilities
386         Consortiums:
387             SampleConsortium:
388                 Organizations:
389                     - *Org1
390                     - *Org2
391                     - *Org3
392     ThreeOrgsChannel:
393         Consortium: SampleConsortium
394         <<: *ChannelDefaults
395         Application:
396             <<: *ApplicationDefaults
397             Organizations:
398                 - *Org1
399                 - *Org2
400                 - *Org3
401             Capabilities:
402                 <<: *ApplicationCapabilities
```

### (2) ./docker/docker-compose-test-net.yaml
set up peer nodes

```
  8 volumes:
  9   orderer.example.com:
 10   peer0.org1.example.com:
 11   peer0.org2.example.com:
 12   peer0.org3.example.com:
``` 
```
 17 services:
...
128   peer0.org3.example.com:
129     container_name: peer0.org3.example.com
130     image: hyperledger/fabric-peer:$IMAGE_TAG
131     environment:
132       #Generic peer variables
133       - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
134       # the following setting starts chaincode containers on the same
135       # bridge network as the peers
136       # https://docs.docker.com/compose/networking/
137       - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_test
138       - FABRIC_LOGGING_SPEC=INFO
139       #- FABRIC_LOGGING_SPEC=DEBUG
140       - CORE_PEER_TLS_ENABLED=true
141       - CORE_PEER_PROFILE_ENABLED=true
142       - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
143       - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
144       - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
145       # Peer specific variabes
146       - CORE_PEER_ID=peer0.org3.example.com
147       - CORE_PEER_ADDRESS=peer0.org3.example.com:6051
148       - CORE_PEER_LISTENADDRESS=0.0.0.0:6051
149       - CORE_PEER_CHAINCODEADDRESS=peer0.org3.example.com:6052
150       - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:6052
151       - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org3.example.com:6051
152       - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org3.example.com:6051
153       - CORE_PEER_LOCALMSPID=Org3MSP
154     volumes:
155         - /var/run/:/host/var/run/
156         - ../organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/msp:/etc/hyperledger/fabric/msp
157         - ../organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls:/etc/hyperledger/fabric/tls
158         - peer0.org3.example.com:/var/hyperledger/production
159     working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
160     command: peer node start
161     ports:
162       - 6051:6051
163     networks:
164       - test
```

### (3) ./docker/docker-compose-ca.yaml
11 services:

```
 45   ca_org3:
 46     image: hyperledger/fabric-ca:$IMAGE_TAG
 47     environment:
 48       - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
 49       - FABRIC_CA_SERVER_CA_NAME=ca-org3
 50       - FABRIC_CA_SERVER_TLS_ENABLED=true
 51       - FABRIC_CA_SERVER_PORT=6054
 52     ports:
 53       - "6054:6054"
 54     command: sh -c 'fabric-ca-server start -b admin:adminpw -d'
 55     volumes:
 56       - ../organizations/fabric-ca/org3:/etc/hyperledger/fabric-ca-server
 57     container_name: ca_org3
 58     networks:
 59       - test
```

### (4) ./scripts/envVar.sh
```
 15 export PEER0_ORG3_CA=${PWD}/organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt

 44   elif [ $USING_ORG -eq 3 ]; then
 45     export CORE_PEER_LOCALMSPID="Org3MSP"
 46     export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG3_CA
 47     export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
 48 #    export CORE_PEER_ADDRESS=localhost:11051
 49     export CORE_PEER_ADDRESS=localhost:6051
```

### (5) ./scripts/createChannel.sh
```
 21 createChannelTx() {
 22 
 23   set -x
 24   configtxgen -profile ThreeOrgsChannel -outputCreateChannelTx ./channel-artifacts/${CHANNEL_NAME}.tx -channelID $CHANNEL_NAME
 ......
 
 33 createAnchorPeerTx() {
 34 
 35   for orgmsp in Org1MSP Org2MSP Org3MSP; do
 36 
 37   infoln "Generating anchor peer update transaction for ${orgmsp}"
 38   set -x
 39   configtxgen -profile ThreeOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/${orgmsp}anchors.tx -channelID $CHANNEL_NAME -asOrg ${orgmsp}
 ......

126 ## Create channel
127 infoln "Creating channel ${CHANNEL_NAME}"
128 createChannel
129 
130 ## Join all the peers to the channel
131 infoln "Join Org1 peers to the channel..."
132 joinChannel 1
133 infoln "Join Org2 peers to the channel..."
134 joinChannel 2
135 infoln "Join Org3 peers to the channel..."
136 joinChannel 3
137 
138 ## Set the anchor peers for each org in the channel
139 infoln "Updating anchor peers for org1..."
140 updateAnchorPeers 1
141 infoln "Updating anchor peers for org2..."
142 updateAnchorPeers 2
143 infoln "Updating anchor peers for org3..."
144 updateAnchorPeers 3
145 
146 successln "Channel successfully joined"
```


### (6) ./scripts/deployCC.sh

```
316 ## package the chaincode
317 packageChaincode
318 
319 ## Install chaincode on peer0.org1 and peer0.org2
320 infoln "Installing chaincode on peer0.org1..."
321 installChaincode 1
322 infoln "Install chaincode on peer0.org2..."
323 installChaincode 2
324 infoln "Install chaincode on peer0.org3..."
325 installChaincode 3
326 
327 ## query whether the chaincode is installed
328 queryInstalled 1
329 
330 ## approve the definition for org1
331 approveForMyOrg 1
332 
333 ## check whether the chaincode definition is ready to be committed
334 ## expect org1 to have approved and org2 not to
335 #checkCommitReadiness 1 "\"Org1MSP\": true" "\"Org2MSP\": false"
336 #checkCommitReadiness 2 "\"Org1MSP\": true" "\"Org2MSP\": false"
337 
338 ## now approve also for org2
339 approveForMyOrg 2
340 ## now approve also for org3
341 approveForMyOrg 3
342 
343 ## check whether the chaincode definition is ready to be committed
344 ## expect them both to have approved
345 checkCommitReadiness 1 "\"Org1MSP\": true" "\"Org2MSP\": true"
346 checkCommitReadiness 2 "\"Org1MSP\": true" "\"Org2MSP\": true"
347 checkCommitReadiness 3 "\"Org1MSP\": true" "\"Org2MSP\": true" "\"Org3MSP\": true"
348 
349 ## now that we know for sure both orgs have approved, commit the definition
350 #commitChaincodeDefinition 1 2
351 commitChaincodeDefinition 1 2 3
352 
353 ## query on both orgs to see that the definition committed successfully
354 queryCommitted 1
355 queryCommitted 2
356 queryCommitted 3
357 
358 ## Invoke the chaincode - this does require that the chaincode have the 'initLedger'
359 ## method defined
360 if [ "$CC_INIT_FCN" = "NA" ]; then
361   infoln "Chaincode initialization is not required"
362 else
363   chaincodeInvokeInit 1 2 3
364 #  chaincodeInvokeInit 1 2
365 fi
```

### (7) ./organizations/ccp-generate.sh 

```
 47 ORG=3
 48 P0PORT=6051
 49 CAPORT=6054
 50 PEERPEM=organizations/peerOrganizations/org3.example.com/tlsca/tlsca.org3.example.com-cert.pem
 51 CAPEM=organizations/peerOrganizations/org3.example.com/ca/ca.org3.example.com-cert.pem
 52 
 53 echo "$(json_ccp $ORG $P0PORT $CAPORT $PEERPEM $CAPEM)" > organizations/peerOrganizations/org3.example.com/connection-org3.json
 54 echo "$(yaml_ccp $ORG $P0PORT $CAPORT $PEERPEM $CAPEM)" > organizations/peerOrganizations/org3.example.com/connection-org3.yaml
```

###  if not use '-ca' option:
(8) create ./organizations/cryptogen/crypto-config-org3.yaml (copy and change from crypto-config-org2.yaml)

(9) ./network.sh

```
169     set -x
170     cryptogen generate --config=./organizations/cryptogen/crypto-config-org3.yaml --output="organizations"
171     res=$?
172     { set +x; } 2>/dev/null
173     if [ $res -ne 0 ]; then
174       fatalln "Failed to generate certificates..."
175     fi
```


