
#Connect to GemFire Redis

	/devtools/repositories/NoSQL/redis-3.2.7/src$ ./redis-cli -h localhost -p 11211


##Example Redis commands

	localhost:11211> HSET server:name "fido"
	OK
	localhost:11211>  GET server:name
	"fido"
	
	localhost:11211> 
	----


Current GeodeRedisAdapter implementation is based on https://cwiki.apache.org/confluence/display/GEODE/Geode+Redis+Adapter+Proposal.
We are looking for some feedback on Redis commands and their mapping to geode region.

==========================
See Pull Request related to GEODE-2469.
The updated Geode Redis Adapter now works with a sample Spring Data Redis Example 

These changes are focused on the HASH and Set Redis Data Types to support Spring Data Redis sample code located at the following URL

	https://github.com/Pivotal-Data-Engineering/gemfire9_examples/tree/person_example_sdg_Tracker139498217/redis/spring-data-redis-example/src/test/java/io/pivotal/redis/gemfire/example/repository

The Hash performance changes from this pull request had a 99.8% performance improvement. 

This is based on the Hashes JUNIT Tests.

	https://github.com/Pivotal-Data-Engineering/gemfire9_examples/blob/person_example_sdg_Tracker139498217/redis/gemfire-streaming-client/src/test/java/io/pivotal/gemfire9/HashesJUnitTest.java

This code executed in 12.549s against Gemfire 9.0.1 code. After the changes, the test executed in 0.022s with the GEODE-2469 pull request.

Redis Set related command performance had a 99.9% performance improvement. 

See  https://github.com/Pivotal-Data-Engineering/gemfire9_examples/blob/person_example_sdg_Tracker139498217/redis/gemfire-streaming-client/src/test/java/io/pivotal/gemfire9/SetsJUnitTest.java

The previous Set Junit tests executed against GemFire 9.0.1 executed in 31.507 seconds. These same test executed in 0.036 seconds with the GEODE-2469 pull request changes.

The GemFire 9.0.1 Geode (1.1.0) version for the Geode Redis adapter created a Geode Region for each key provided in the Redis Hash or Set related command. Each Redis command to remove key entry previously destroyed the region. The big performance gain is based on using a new ReDiS_HASH and ReDiS_SET region. Note the changed will create or reuse an existing region with a matching name for Redis HASH commands references objects. For Redis HASH object's key will have a format of object:key

Please see https://redis.io/topics/data-types-intro HASH section on objects for information on Redis objects.











==========================
  
## Hashes
 
Based on my Spring Data Redis testing for Hash/object support.

HMSET and similar Hash commands are submitted in the following format: HMSET region:key [field value]+
I proposed creating regions with the following format:

Region<ByteArrayWrapper,Map<ByteArrayWrapper,ByteArrayWrapper>> region;

Also see Hashes section at the following URL https://redis.io/topics/data-types

Example Redis command:
HMSET companies:100 _class io.pivotal.redis.gemfire.example.repository.Company id 100 name nylaInc email info@nylaInc.io website nylaInc.io taxID id:1 address.address1 address1 address.address2 address2 address.cityTown cityTown address.stateProvince stateProvince address.zip zip address.country country  =>

	//Pseudo Access code
	Region<ByteArrayWrapper,Map<ByteArrayWrapper,ByteArrayWrapper>> companiesRegion = getRegion("companies")
	companiesRegion.put(100, toMap(fieldValues))
	
//------
// HGETALL region:key

HGETALL companies:100 =>
	
	Region<key,Map<field,value>> companiesRegion = getRegion("companies")
	return companiesRegion.get(100)

//HSET region:key field value

HSET companies:100 email updated@pivotal.io =>

	Region<key,Map<field,value>> companiesRegion = getRegion("companies");
	Map map = companiesRegion.get(100)
	map.set(email,updated@pivotal.io)
	companiesRegion.put(100,map);

## Set

The proposal is to use a single partition region (ex: __RedisSET) for the SET commands.

Region<byteArrayWrapper,HashSet<byteArrayWrapper>> region;

Example Redis commands

SADD myset "Hello" =>

	Region<ByteArrayWrapper,Set<ByteArrayWrapper>> region = getRegion("__RedisSET");
	Set set = region(myset)
	boolean bool = set.add(Hello)
	if(bool){
	  region.put(myset,set)
	}
	return bool;


SMEMBERS myset "Hello" =>

		Region<ByteArrayWrapper,Set<ByteArrayWrapper>> region = getRegion("_RedisSET");
			Set set = region(myset)
			return set.contains(Hello)s  


=============
# Testing

The Spring Data Redis test example were verified with the following GEODE forked branch.

[https://github.com/ggreen/geode/tree/GEODE-2469](https://github.com/ggreen/geode/tree/GEODE-2469)

A pull request as been submitted for these changes and is pending review.
