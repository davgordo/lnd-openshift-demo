# Lightning Network demo for OpenShift

This repository contains a manifest for deploying the [Lightning Network Daemon Docker demo](https://github.com/lightningnetwork/lnd/tree/master/docker) to OpenShift.

## Instructions

Follow these steps to deploy the demo.

### Create a project for the demo

`oc new-project lightning`

### Prepare RPC server certificates

#### Install the cert-manager operator

Go to the OperatorHub and install the community version of cert-manager for all namespaces.

#### Generate certificates

Deploy a cert-manager instance:

`oc apply -f certificates/CertManager_cert-manager.yaml`

Wait until all cert-manager pods are running:

`oc get pods -w`

Apply the certificate manifest to your target namespace:

`oc apply -f certificates/`

The RPC certificate data is now stored in a secret called `btcd-rpc-tls`

Verify that the secret exists and contains 3 keys (this may take a minute or two):

`oc describe secret btcd-rpc-tls`

### Build `btcd` and `lnd` container images

Apply the build manifests to your target namespace:

`oc apply -f build/`

Build the container images:

`oc start-build btcd --follow`

`oc start-build lnd --follow`

### Deploy the Bitcoin node

Apply the `btcd` deployment manifest:

`oc apply -f btcd/`

Verify the daemon is started successfully:

`oc rsh deployment/btcd ./start-btcctl.sh getblockchaininfo`

You should see a JSON response that indicates there are zero blocks.

### Deploy Alice and Bob's Lightning nodes

Apply the deployment manifest for Alice:

`oc apply -f lnd/`

### Make Alice rich

Create a new address for Alice:

`oc rsh deployment/alice lncli --network=simnet newaddress np2wkh`

This will return a JSON response that contains Alice's new address.

Set the `MINING_ADDRESS` environment variable on the `btcd` deployment with the value from the previous step:

`oc set env deployment/btcd MINING_ADDRESS=<alice address>`

This should redeploy the `btcd` pod.

Mine 400 blocks (the reward will go to Alice's address):

`oc rsh deployment/btcd ./start-btcctl.sh generate 400`

Verify that the previous step activated segwit:

`oc rsh deployment/btcd ./start-btcctl.sh getblockchaininfo`

Now Alice is rich, let's make sure:

`oc rsh deployment/alice lncli --network=simnet walletbalance`

Alice's balance should show she's been stacking Sats.

### Make Alice and Bob transact

Find Bob's pub key:

`oc rsh deployment/bob lncli --network=simnet getinfo`

Connect Alice to Bob:

`oc rsh deployment/alice lncli --network=simnet connect <bob's pub key>@bob`

Verify Alice has a peer:

`oc rsh deployment/alice lncli --network=simnet listpeers`

Open a channel between Alice and Bob:

`oc rsh deployment/alice lncli --network=simnet openchannel --node-key=<bob's pub key> --local_amt=1000000`

Mine some blocks to activate the channel:

`oc rsh deployment/btcd ./start-btcctl.sh generate 3`

Verify that the channel is active:

`oc rsh deployment/alice lncli --network=simnet listchannels`

Generate an invoice from Bob:

`oc rsh deployment/bob lncli --network=simnet addinvoice --amt=10000`

This will return a JSON response that contains an encoded version of the invoice as the value of the `payment_request` attribute.

Pay Bob's invoice from Alice's address:

`oc rsh deployment/alice lncli --network=simnet sendpayment --pay_req=<encoded invoice>`

Verify Bob has a channel balance:

`oc rsh deployment/bob lncli --network=simnet channelbalance`

Get the funding transaction ID and the output index of the channel:

`oc rsh deployment/alice lncli --network=simnet listchannels`

This will return a JSON response with an attribute called `channel_point`. The value of that attribute is in the form of `<funding_txid>:<output_index>`.

Close the channel using output values of the previous step:

`oc rsh deployment/alice lncli --network=simnet closechannel --funding_txid=<funding_txid> --output_index=<output_index>`

Mine a few more blocks to finalize:

`oc rsh deployment/btcd ./start-btcctl.sh generate 3`

Bob should have some Sats now:

`oc rsh deployment/bob lncli --network=simnet walletbalance`
