beacon chain

每个slot（时隙，时段）是6秒。每个beacon chain的beaconblock结构中有一个slot字段，可视为beacon chain中的block height。



BeaconBlock的头(Header)里面得有pow链的收据根

prysm使用boltdb（嵌入式kv数据库）——数据层