# Introduction

Fabric v2.0 supports chaincode deployment and execution outside of Fabric that enables users to manage a chaincode runtime independently of the peer. This really hrlps in the deployment of chaincode on Fabric in deployments such as Kubernetes. This becomes really importent because of the current plan of kubernetes to removing docker runtime. we will not be able to use the docker-in-docker option which peer uses to deploy chaincode. So, instead of building and launching the chaincode on every peer, chaincode can now run as a service whose lifecycle is managed outside of Fabric. This capability leverages the Fabric v2.0 external builder and launcher functionality which enables operators to extend a peer with programs to build, launch, and discover chaincode.

# Running fabcar Node.js chaincode as an external service

Setting up the external builder and launcher

We will need to customize our peer first to enable this feature. In this tutorial, we use very simple shell scripts that can be run directly within the default Fabric peer images.
Modify the externalBuilders field in the core.yaml file [inside config folder] to resemble the configuration below:

Modify the externalBuilders field in the core.yaml file to resemble the configuration below:

```shell
externalBuilders:
    - path: /opt/gopath/src/github.com/hyperledger/fabric-samples/fabcar/chaincode-external/sampleBuilder
      name: external-sample-builder
```

This update sets the name of the external builder as external-sample-builder, and the path of the builder to the scripts provided in this sample. Note that this is the path within the peer container, not your local machine.

To set the path within the peer container, you will need to modify the docker compose file to mount a couple of additional volumes. Open the file test-network/docker/docker-compose-test-net.yaml, and add to the volumes section of both peer0.org1.example.com and peer0.org2.example.com the following two lines:

```shell
- ../..:/opt/gopath/src/github.com/hyperledger/fabric-samples
- ../../config/core.yaml:/etc/hyperledger/fabric/core.yaml
```

This update will mount the core.yaml that we modified into the peer container and override the configuration file within the peer image. The update also mounts the fabric-sample builder so that it can be found at the location that you specified in core.yaml. You also have the option of commenting out the line - /var/run/docker.sock:/host/var/run/docker.sock, since we no longer need to access the docker daemon from inside the peer container to launch the chaincode.

# Packaging and installing Chaincode

The fabcar external chaincode requires two environment variables to run, CHAINCODE_SERVER_ADDRESS and CHAINCODE_ID, which are described and set in the chaincode.env file.

You need to provide a connection.json configuration file to your peer in order to connect to the external fabric service. The address specified in the connection.json must correspond to the CHAINCODE_SERVER_ADDRESS value in chaincode.env, which is fabric.org1.example.com:9999 in our example.

Because we will run our chaincode as an external service, the chaincode itself does not need to be included in the chaincode package that gets installed to each peer. Only the configuration and metadata information needs to be included in the package. Since the packaging is trivial, we can manually create the chaincode package.

Open a new terminal and navigate to the chaincode/fabcar/typescript directory.

```shell
cd chaincode/fabcar/typescript
```

First, create a code.tar.gz archive containing the connection.json file:

```shell
tar cfz code.tar.gz connection.json
```

Then, create the chaincode package, including the code.tar.gz file and the supplied metadata.json file:

```shell
tar cfz fabcar.tgz metadata.json code.tar.gz
```

Now, we are ready to deploy the chaincode.

# Starting the test-network

We will use the Fabric test network to run the external chaincode. Open a new terminal and navigate to the test-network directory.

```shell
  cd test-network
```

Run the following command to deploy the test network and create a new channel:

```shell
./network.sh up createChannel -ca -c mychannel
```

We are now ready to deploy the external chaincode.

# Installing the external chaincode

From the test-network directory, set the following environment variables to use the Fabric binaries:

Installing the external chaincode

```shell
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=$PWD/../config/
```

Run the following command to import functions from the envVar.sh script into your terminal. These functions allow you to act as either test network organization.

```shell
. ./scripts/envVar.sh
```

Run the following commands to install the fabcar.tar.gz chaincode on org1. The setGlobals function simply sets the environment variables that allow you to act as org1 or org2.

Install the chaincode package on the org1 peer:

```shell
setGlobals 1
peer lifecycle chaincode install ../chaincode/fabcar/typescript/fabcar.tgz
```

Install the chaincode package on the org2 peer:

```shell
setGlobals 2
peer lifecycle chaincode install ../chaincode/fabcar/typescript/fabcar.tgz
```

Query installed chaincode package id

```shell
setGlobals 1
peer lifecycle chaincode queryinstalled --peerAddresses localhost:7051 --tlsRootCertFiles organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
```

The command will return output similar to the following:

```shell
Installed chaincodes on peer:
Package ID: fabcar_1.0:25398f34cf1667abe92bd498dad2ceb882a055417a5ad21e5d264cd2ec14c11a, Label: fabcar_1.0
```

# Running the fabcar external service

We are going to run the smart contract as an external service by first building and then starting a docker container. Open a new terminal and navigate back to the /chaincode/fabcar/typescript directory:

In this directory, open the chaincode.env file to set the CHAINCODE_ID variable to the same package ID that was returned by the peer lifecycle chaincode queryinstalled command. The value should be the same as the environment variable that you set in the previous terminal. Since, we are using fabric-contract-api for chaincode, I have modified the package.json as shown below:

```shell
   "scripts": {
        "lint": "tslint -c tslint.json 'src/**/*.ts'",
        "pretest": "npm run lint",
        "test": "nyc mocha -r ts-node/register src/**/*.spec.ts",
        "start": "fabric-chaincode-node server --chaincode-address fabcar.org1.example.com:9999 --chaincode-id fabcar_1.0:25398f34cf1667abe92bd498dad2ceb882a055417a5ad21e5d264cd2ec14c11a",
        "build": "tsc",
        "build:watch": "tsc -w",
        "prepublishOnly": "npm run build"
    }
```

After you edit the chaincode.env file, you can use the Dockerfile to build an image of the external fabcar chaincode:

```shell
docker build -t hyperledger/fabcar .
```

Then run the image to start the fabcar service:

```shell
docker run -it --rm --name fabcar.org1.example.com --hostname fabcar.org1.example.com --env-file chaincode.env --network=fabric_test hyperledger/fabcar
```

This will start and run the external chaincode service within the container.

# Deploy the Fabcar external chaincode definition to the channel

Navigate back to the test-network directory to finish deploying the chaincode definition of the external smart contract to the channel. Make sure that your environment variables are still set.

```shell
export CHAINCODE_ID=fabcar_1.0:25398f34cf1667abe92bd498dad2ceb882a055417a5ad21e5d264cd2ec14c11a

setGlobals 2
peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" --channelID mychannel --name fabcar --version 1.0 --package-id $CHAINCODE_ID --sequence 1

setGlobals 1
peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" --channelID mychannel --name fabcar --version 1.0 --package-id $CHAINCODE_ID --sequence 1

peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" --channelID mychannel --name fabcar --peerAddresses localhost:7051 --tlsRootCertFiles "$PWD/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" --peerAddresses localhost:9051 --tlsRootCertFiles organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --version 1.0 --sequence 1
```

The commands above approve the chaincode definition for the external chaincode and commits the definition to the channel. The resulting output should be similar to the following:

```shell
2021-07-17 15:41:16.131 WEST [chaincodeCmd] ClientWait -> INFO 001 txid [0d9cb0a14c1d0294646af7c9f4bc9e053e53d8bb9d1ca67de56ed56e3db32cd4] committed with status (VALID) at localhost:7051
2021-07-17 15:41:16.159 WEST [chaincodeCmd] ClientWait -> INFO 002 txid [0d9cb0a14c1d0294646af7c9f4bc9e053e53d8bb9d1ca67de56ed56e3db32cd4] committed with status (VALID) at localhost:9051
```

# Run Fabcar application

Open yet another terminal and navigate to the fabric-samples/asset-transfer-basic/application-javascript directory:

```shell
cd /fabcar/javascript
```

Register an admin user

```shell
node enrollAdmin.js
```

Register an application user

```shell
node registerUser.js
```

I have modified the invoke.js and query.js file to use the correct chaincode. This file invokes the initLedger function of the chaincode.

```shell
node invoke.js
```

To get the list of all the registered car, run query.js.

```shell
node query.js
```
