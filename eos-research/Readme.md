
# Integration of EOS node with Kafka

Some notes on how to integrate a EOS node with Kafka

## EOS compilation

The compilation was done as described in:

https://developers.eos.io/eosio-nodeos/docs/docker

It uses docker compose to build an eosio/eos image using the exising eosio/builder image. It is considered the fastest/easiest option as the builder image is already correctly set up.

## Kafka plugin compilation

The Kafka plugin from:

https://github.com/TP-Lab/kafka_plugin

is based on the MongoDB plugin and appears to work. If can be built after the rest of the binaries. The approach I chose, was to bring up a eosio/builder container and use it for re-builds. Then I transferred the newly built nodeos binary to the running eosio/eos container. EOS plugins are built as static libraries and linked in the main node binary at link time. The build of the plugin itself is straightforward using the provided cmake files and the short description on their GitHub.

## Running Kafka

I used the Santiment compose file provided at:

https://github.com/santiment/san-exporter

it brings up both the Kafka and Zookeeper containers.

I created the needed topics using:

```shell
/opt/kafka_2.11-2.0.0/bin/kafka-console-consumer.sh --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 1 --topic eos_applied_topic
/opt/kafka_2.11-2.0.0/bin/kafka-console-consumer.sh --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 1 --topic eos_accept_topic
```

from within the Kafka container.

## Connecting the plugin to Kafka

Firstly connect the nodeos container to the kafka/zookeeper docker network with:

```shell
docker network connect nodeos_container_id kafka_network_id
```

once the two containers are in the same network start nodeos with:

```shell
/opt/eosio/bin/nodeos --config-dir=/opt/eosio/bin/data-dir --genesis-json=/opt/eosio/bin/data-dir/genesis.json --plugin eosio::kafka_plugin --kafka-uri 172.20.0.3:9092 --accept_trx_topic eos_accept_topic  --applied_trx_topic eos_applied_topic --kafka-block-start 100 --kafka-queue-size 5000
```

genesis.json should match the Mainnet genesis available in Internet, otherwise the other peers would reject the connection. List of peers can be obtained from example from:
https://eosnodes.privex.io/

The peers need to be pasted in config.ini like so:

```
p2p-peer-address = api-full1.eoseoul.io:9876
p2p-peer-address = api-full2.eoseoul.io:9876
p2p-peer-address = api.eosuk.io:12000
p2p-peer-address = boot.eostitan.com:9876
p2p-peer-address = bp.antpool.com:443
```

I also had to comment out a few options to prevent block production on mainnet:

```
#enable-stale-production = true
#producer-name = eosio
```


## Data collected

Using the following commands we inspect how the two topics are populated:

```
bash-4.4# /opt/kafka_2.11-2.0.0/bin/kafka-console-consumer.sh --max-messages 3 --bootstrap-server localhost:9092 --topic eos_accept_topic --from-beginning
{"expiration":"2018-06-09T11:56:30","ref_block_num":1,"ref_block_prefix":4126519930,"max_net_usage_words":0,"max_cpu_usage_ms":0,"delay_sec":0,"context_free_actions":[],"actions":[{"account":"eosio","name":"onblock","authorization":[{"actor":"eosio","permission":"active"}],"data":"d1eb59450000000000000000010000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906000000000000"}],"transaction_extensions":[],"signatures":[],"context_free_data":[]}
{"expiration":"2018-06-09T11:56:31","ref_block_num":2,"ref_block_prefix":976177227,"max_net_usage_words":0,"max_cpu_usage_ms":0,"delay_sec":0,"context_free_actions":[],"actions":[{"account":"eosio","name":"onblock","authorization":[{"actor":"eosio","permission":"active"}],"data":"dcf95c450000000000ea3055000000000001405147477ab2f5f51cda427b638191c66d2c59aa392d5c2c98076cb00000000000000000000000000000000000000000000000000000000000000000e0244db4c02d68ae64dec160310e247bb04e5cb599afb7c14710fbf3f4576c0e000000000000"}],"transaction_extensions":[],"signatures":[],"context_free_data":[]}
{"expiration":"2018-06-09T11:56:31","ref_block_num":3,"ref_block_prefix":3833319509,"max_net_usage_words":0,"max_cpu_usage_ms":0,"delay_sec":0,"context_free_actions":[],"actions":[{"account":"eosio","name":"onblock","authorization":[{"actor":"eosio","permission":"active"}],"data":"ddf95c450000000000ea305500000000000267f3e2284b482f3afc2e724be1d6cbc1804532ec62d4e7af47c3069300000000000000000000000000000000000000000000000000000000000000001fb815fab36fcb7abda71a209c804db4d7d5c20c164035344f7efecb608d6b53000000000000"}],"transaction_extensions":[],"signatures":[],"context_free_data":[]}
Processed a total of 3 messages
```

```
/opt/kafka_2.11-2.0.0/bin/kafka-console-consumer.sh --max-messages 3 --bootstrap-server localhost:9092 --topic eos_applied_topic --from-beginning
{"block_number":101,"block_time":1528545439500,"trace":{"id":"289a029400c4b339111a66f9dc07be0732a440234f17d966daea2984658dcf47","block_num":101,"block_time":"2018-06-09T11:57:19.500","producer_block_id":"00000065c1ca7aeca286bb49acb2fc0c52633f141dc6600186c2c7f707d087a9","receipt":{"status":"executed","cpu_usage_us":100,"net_usage_words":0},"elapsed":177,"net_usage":0,"scheduled":false,"action_traces":[{"receipt":{"receiver":"eosio","act_digest":"81010135df36c6774a7ae2ff820ac9a46011b75ad718425ff5d1834c07cfaed3","global_sequence":100,"recv_sequence":100,"auth_sequence":[["eosio",100]],"code_sequence":0,"abi_sequence":0},"act":{"account":"eosio","name":"onblock","authorization":[{"actor":"eosio","permission":"active"}],"data":"3efa5c450000000000ea30550000000000636603d20fb205862740008ca9558ee005465a546ea6ec346a2a5e055d00000000000000000000000000000000000000000000000000000000000000006c352258b8461adbe770f006457f879e5ce2038459d4f95943b2ecfc5ff5a28e000000000000"},"context_free":false,"elapsed":50,"console":"","trx_id":"289a029400c4b339111a66f9dc07be0732a440234f17d966daea2984658dcf47","block_num":101,"block_time":"2018-06-09T11:57:19.500","producer_block_id":"00000065c1ca7aeca286bb49acb2fc0c52633f141dc6600186c2c7f707d087a9","account_ram_deltas":[],"inline_traces":[]}],"failed_dtrx_trace":null}}
{"block_number":102,"block_time":1528545440000,"trace":{"id":"f271bb4f839e4a4d5500c1803b218f9ec97f8606fea82b04f017210bb4070156","block_num":102,"block_time":"2018-06-09T11:57:20.000","producer_block_id":"000000668474bc46fb3cddee286874f52071e26e554e2acd040ba89bb2bc9110","receipt":{"status":"executed","cpu_usage_us":100,"net_usage_words":0},"elapsed":283,"net_usage":0,"scheduled":false,"action_traces":[{"receipt":{"receiver":"eosio","act_digest":"da7768d00376370b8f7dbe51845324acebe5203a20681dba82b7dc2e8b142ab5","global_sequence":101,"recv_sequence":101,"auth_sequence":[["eosio",101]],"code_sequence":0,"abi_sequence":0},"act":{"account":"eosio","name":"onblock","authorization":[{"actor":"eosio","permission":"active"}],"data":"3ffa5c450000000000ea305500000000006492871283c47f6ef57b00cf534628eb818c34deb87ea68a3557254c6b0000000000000000000000000000000000000000000000000000000000000000cf8b585449e982e69afdafff411b5775eee4e9b6e0942caff41adfbbdaa18cc9000000000000"},"context_free":false,"elapsed":50,"console":"","trx_id":"f271bb4f839e4a4d5500c1803b218f9ec97f8606fea82b04f017210bb4070156","block_num":102,"block_time":"2018-06-09T11:57:20.000","producer_block_id":"000000668474bc46fb3cddee286874f52071e26e554e2acd040ba89bb2bc9110","account_ram_deltas":[],"inline_traces":[]}],"failed_dtrx_trace":null}}
{"block_number":103,"block_time":1528545440500,"trace":{"id":"0ea7e4c97ac2f1316acb959d931e8dbfee24d546562cf094dc8cea1a83e1812c","block_num":103,"block_time":"2018-06-09T11:57:20.500","producer_block_id":"000000674400d832037266988e187bb4835f29bec570a9e57820fe3baa93ce54","receipt":{"status":"executed","cpu_usage_us":100,"net_usage_words":0},"elapsed":88,"net_usage":0,"scheduled":false,"action_traces":[{"receipt":{"receiver":"eosio","act_digest":"32d7d481b0eff7ec826823629af4af5de5f96886336eaa9c5aeae0edf9ed1bb4","global_sequence":102,"recv_sequence":102,"auth_sequence":[["eosio",102]],"code_sequence":0,"abi_sequence":0},"act":{"account":"eosio","name":"onblock","authorization":[{"actor":"eosio","permission":"active"}],"data":"40fa5c450000000000ea3055000000000065c1ca7aeca286bb49acb2fc0c52633f141dc6600186c2c7f707d087a90000000000000000000000000000000000000000000000000000000000000000f722a941bd8dd4bbecd51ddb9986f97815691e1937499f0ec7138410e77194ef000000000000"},"context_free":false,"elapsed":22,"console":"","trx_id":"0ea7e4c97ac2f1316acb959d931e8dbfee24d546562cf094dc8cea1a83e1812c","block_num":103,"block_time":"2018-06-09T11:57:20.500","producer_block_id":"000000674400d832037266988e187bb4835f29bec570a9e57820fe3baa93ce54","account_ram_deltas":[],"inline_traces":[]}],"failed_dtrx_trace":null}}
Processed a total of 3 messages
```

## C++ code analysis

The Kafka plugin receives its data from "class controller" (as well as all the other plugins). Currently the Kafka plugin is interested in 'accepted transactions' and also 'applied transactions' which matches the two topics given in the example. Other methods are left empty and can easily be activated.

```C++
kafka_plugin_impl::_process_accepted_block()
kafka_plugin_impl::_process_irreversible_block()
```

However even more information can be obtained from the controller.

The Kafka plugins sends its data to the Kafka cluster in:

```C++
 kafka_plugin_impl::consume_blocks()
 ```

 this method is handled in a separate thread. The queues are being populated using the controller thread. The plugin attaches to the controller in the method

```C++
 kafka_plugin::plugin_initialize()
 ```
