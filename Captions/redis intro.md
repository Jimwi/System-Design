so hello and welcome to the first of what we hope will be many deep Dives on core Technologies. 
today we're going to be talking about redis. I'm Stefan I am one of the co-founders of hello interview I've conducted north of 2,000 interviews in my career. 
and our hope with these sessions is to give you some of the depth necessary to be successful in system design interviews.

system design is certainly about solving problems from end to end 
but the interviewers in these sessions are going to be asking you questions to try to test your knowledge
to make sure that you don't just have a cursory understanding of the concepts and the technologies that are involved. 

and so what we're trying to do here is to give you some of that Foundation or at least give you a start to that Foundation if you don't have it and reddis is a great starting point

so why are we learning about reddis to begin with 

well redis is an incredibly versatile technology you can use it in a number of different circumstances from caches distrib locks leaderboards it can be used as a replacement for Kafka in certain instances 

and so there's a lot of bang for your buck with learning about redus and how it can apply to various system design problems 

the second reason that we really prefer redus in a lot of circumstances is because it's very simple to understand 

the conceptual model of redus is actually quite simple and that means that you can both explain your choices but also understand their implications 

if you need to dive into the details of how a SQL query planner is operating in postgress. 
forget it you've got to learn 30 years of history in order to get there.


meanwhile if you need to know why a rdus query is slow it's actually pretty straightforward for you to reason through 
and so redus and particularly this session is hopefully a really good return on your time 

so let's talk about redis starting with what it looks like from a enduser or developer perspective 

then we'll talk about how it actually oper Ates underneath the covers 

and finally we'll merge the two to tell you about how you can use it in various system design scenarios and some of the pitfalls that you might encounter uh in those uh various problems 

so the first thing that you should know about redus is it is a single-threaded inmemory data structure server 

all of those words actually mean something 
single-threaded is unusual to hear in distributed systems mostly because it fails to utilize multicore systems 

but it actually simplifies things quite a lot 
in many databases the order of operations is hard to uh Gro 
and in redus it's quite simple the first request that comes in is the first one that actually runs and everybody else Waits 

this has some dramatic implications for how you actually use redus but it's a great simplifier for how it's working under the covers 

the second thing to know about redus is it is in memory that means it is lightning fast and and can operate in sub millisecond times especially for simple operations like sets and gets 

it also has some implications for how you use it in your system you can't necessarily guarantee the durability of data uh 

but it means you can use redus in places that you might not others with a SQL database you need to make sure that you batch all of your requests together or you run risks of that n plus1 problem 

with reddis you can fire off 1000 requests and the server will happily return its results to you 

it it really changes how you think about operating with a database 

the last piece is that reddis is a data structure server what does that mean well the core of redus is this kind of key value dictionary but Rd values don't necessarily need to be strings or numbers or binary blobs they can also be sorted sets or hashes or geospatial indexes or Bloom filters 

the way that reddis operates is trying to mirror what you might think of from a data structures and algorithms course that you learn about a bunch of really primitive data structures and reddis just gives you a way to use them in a distributed fashion which can be super convenient because it means you can take all of the knowledge that you've accumulated in building up data structures and algorithms expertise and 
lift it into a system design context which is quite nice 

now how does redus actually operate it's really simple you can Brew install redus on your laptop and actually get a CLI that you can use and the wire protocol is very basic it accepts commands like what you see here 

so I can call set and I set it I call get and I get it I call incor and it increments it atomically 

each of these operates on a key remember that the core of redus is this dictionary of keys to these values and the commands are basic basically ways to manipulate the various data structures if you look in the Rus documentation they are organized by the type of the data so it wouldn't make sense for instance to call incor on a hash but there's another command that will allow you to increment the values of a hash 

now these commands can get pretty sophisticated so here's an example of a way to add an item to a stream that's not really important just yet the important thing is that you have a command type in this case the ad operates on a stream that's what the X prefix is and then there's the key that's where it's operating 

now why do the keys matter well the keys are really how redus handles uh the multi- node environments so you can just run redus on a single node it runs in a single thread and it can basically write the commands that successfully execute out to dis you can configure the interval the default I think is every 1 second which basically means redis can lose some data but in the ideal scenario what will happen if the process fails is redus will go down and when it comes back up it will read from that file and recover somewhat gracefully in practice this is horrible 

so for most people what they're going to do is actually have a replica and the way that this works is you have some main or master and then you have a secondary that's reading from that append only file or basically from the log this works much like change data capture when a command runs successfully here it gets it gets run on the secondary there is some special behavior in redus where if there's been 5 minutes or an hour and I haven't gotten an update then the secondary will basically fully rebuild from the contents of uh the master or main but in most instances they're going to and ideally they're going to be caught up to one another 
but that's not very interesting that really constrains us to the throughput of of this single instance we can I guess technically scale our readr put indefinitely if we want to put a bunch of replicas 

but how do we scale out rights and this is where the keyspace starts to come into the picture 

so redus has an internal concept they call a slot which is basically a hash of the key I think it's usually a CRC modulo some number 16 384 I think 
and that is the slot that that key occupies 
theoretically when the cluster isn't resizing a single master or main will own that slot and clients should be aware of all of the nodes in the cluster 
so if I have a request for Foo I'm going to take the hash of Foo the key I'm going to look up the slot that it occupies and then decide which node in the cluster I need to route my request to 

each of the nodes in the clust ER communicates via a gossip protocol so they all know about the existence of each other as well as which keys they have which slots they have and 

if you make a request to the wrong host it will tell you that key doesn't exist here it's been moved 

but for performance sake it's way more beneficial if the client knows exactly which host to go to and so that's why when you start up a client in redus you make it aware of all of the hosts that are available 

you the ically aren't making requests that bounce across uh many different notes and 

so this is where the keyspace becomes incredibly important 

the only way to Shard redus is through choosing your keys and then when you are choosing how to Shard you're functionally choosing how to spread your data across the keyspace 

so let's take a canonical example for a moment if you have a cache one of the major difficulties that you usually have is what's called a hot key problem where many of your requests are going to the same key well how does this break reddis if one of your keys is located on this node and the aggregate traffic to this node exceeds what this node can handle then it doesn't matter that these other hosts are out there the uneven distribution of traffic is going to kill you so what can you do well with redis one of the very common pattern is to Simply append a random number to the key so that way you write the key to multiple hosts and you can read it from multiple hosts so this provides a really crude way to distribute the load across the cluster but if you're thinking about how you scale redus you should be thinking about your key space this is really essential to how you reason about how uh redus scales 

okay so let's walk through some common instances of using red in actual designs and talk about the types of things that might come up in your deep Dives and conversations with your interviewer 


so the first use of redus which is the most common is to use it like a cache and the usage here is pretty simple but let me just explain it quickly you have a database where you need to make heavy queries maybe they're analytic queries maybe this is a deep search index whatever and you want to be able to scale this so what do you do well we're going to create a redis cachee off to the side and our service is first going to check the cache it's going to be very very fast see if there's an entry in there for whatever query we're going to make if there isn't we're going to go to the database grab it and then store the result in the cache if there is we're just going to go and take the result this is appropriate in basically any case where you can tolerate some staleness or inconsistency this is great when you have an eventually consistent system and in a lot of cases that's appropriate there are some places where this obviously doesn't matter now you set up a cash like this you've really got two concerns that you need to address the first is the hotkey issue which I just talked about earlier and so we need to make sure that our cash is spreading out the load amongst all of the instances as much as possible the way we do this in rdus is by assigning keys and so we might append values to our keys such that we are evenly Distributing the requests across all of our redus cache 

the next thing that you need to be consider concerned with is expiration and a common question in interviews is what's the expiration policy of your cash there's a bunch of different options here and reddis fundamentally supports them all the most common way to use reddis is to use the expire command or to basically attach arguments to your sets and gets and what that does is after a certain time that item won't be readable and 

so basically you can say expire after you know 60 seconds if that's your cash time to live and then you can guarantee that anyone reading out of your cash won't read that stale data 

another way to configure R redus is in its uh least recently used setup and this works a little bit differently the way that the least recently used setup Works inside redis is basically you'll continue to be able to append Keys into your cache and functionally indefinitely until you run out of memory and then at that point redis will start to evict those least recently used Keys uh from your setup 

works very much like memcache and in many cases there a drop in replacement but a reasonable Choice 

another use of reddis which is quite simple and quite popular is to use it as a rate limiter 

the way this works is we want to guard this expensive service from getting lots of requests maybe our Downstream has decided that they can only accept five requests every 60 seconds so what we need to do is make sure that if we have multiple instances of this service that they aren't all making in aggregate more than five requests over 60 seconds so how do we do this well I talked about the atomic increment command earlier on in the session and what it does is it increments a key if it exists if it doesn't exists it'll set it to one and it will return the final value think of it like plus plus variable if you're familiar with C++ or or other languages that support that syntax 

the idea is if this value is over the limit in my case five then I don't want to make that request I need to wait on the other hand if it's under that that means that I've got space and I can proceed with the request 

and then the next thing I'm going to do if I actually get an opportunity to make that request is I'm going to make sure that this key gets expired so I'm going to say that I need to expire this key after 60 seconds and what this is going to do in practice is it's going to allow up to end requests to proceed through and then after however many seconds in this case I've set it to a minute has finished then that key gets automatically zeroed out and subsequent Services can make a request again now there is some mechanics that you'll have to insert here the service basically needs to make redundant calls um after it gets rate limited because needs to check in again when the rate limit has expired and this also doesn't behave particularly well when the rate limits are under extreme stress so if I had maybe 60 requests that I needed to make it means that all of my services are going to be hitting redit at the same time and I don't have any specific ordering that is going to be enforced so I might starve one service 

so there's a lot to talk about here in a system design interview between how you Asser ass ERT fairness of your services how you set the limits what's most appropriate and this is the most basic implementation of a rate limiter there's a lot of other structures that you could potentially use that include windows and give clients some idea about when they might be next in line so on and so forth so design your rate limiter as you see fit but keep in mind there's a lot of logistics that can go into this depending upon the needs of your system sometimes something simple is great so far we've talked mostly about use cas cases of redus that rely on its key value nature maybe with some expiration mechanics but the value of reddis and its data structure server is primarily that these data structures can become increasingly powerful and I want to talk about a few instances of that 

the most powerful and honestly the most complicated primitive that I think redus offers is its stream and 
the idea behind a stream is pretty simple there was actually a Kafka paper that was written I think by LinkedIn many years ago that talks about the of distributed append only logs in system design 

so you can imagine Redd streams as basically order lists of items they're given an ID which is usually a Timestamp as they're inserted and each of these items can have their own keys and values think of them like Json objects or hashes and 

the nice thing about this stream is if we've got something that we need to make sure all of the items of the stream are processed then rdus gives us a bunch of of Primitives to work with us

so let's talk about if I needed to build an async job queue 
so what I want to be able to do is insert items onto that queue and have them processed in order and reliably 
I want to make sure that if an item is uh inserted onto the queue that it will eventually get processed 

what I'm going to do for storing these items is I'm going to create a stream 
I'll put each of those items on the stream as they're created 
and then next I'm going to create this thing called a consumer group 
you can think of a consumer group like a pointer into the stream and that pointer basically defines where we're at and so a consumer group will keep track of where in the Stream it has to keep processing 

the workers can each query the consumer group for unallocated items 
so if the consumer group happens to be pointing at item two and no workers have picked that one up then that item is going to be allocated to that worker 

there's a final notion within streams and consumer groups around what redus calls claiming and the idea is at any given moment only one of these workers can have claimed an item on the consumer group 

and if for instance the worker fails which it's apt to do then that item can be reclaimed by the consumer group and allocated out to an additional worker 

so the idea behind a reddish stream is it gives you a way to distribute work amongst many workers in a way that's fault tolerant partially because you you have all that normal caveats about redis and that is very um very fast and so you don't have to have a bunch of latency that you insert into the process 

now what does this mean in the system design setting well there are a bunch of considerations that you should figure out before you go and implement this the first is is you need to be able to handle failures of redus and there are options like using a fork of redus like memory DB that will give you more reliability you also can build some redundancy in by having replications or additional shards basically by additional Keys 

you'll also want to figure out how you can keep these workers allocated the right work 
the typical way this is accomplished is that the worker while it's processing an item will continue to heartbeat back to the consumer group 

letting it know hey I'm still working on this and that way the consumer group isn't snatching back the work item uh before it's had a chance to finish it 

but there are definitely scenarios in distributed system where this doesn't work as an example if worker 2 loses network connectivity to the consumer group but maybe still has it to a database or a downstream system it might continue to process that item while the consumer group reclaims it and hands it off to another worker 

so the behavior here is that your consumer group is going to offer at least once guarantees but it's not going to guarantee its process exactly once and in some cases that may be exactly what you need 

let's talk about two more data structures that are really helpful in system design contexts for this application let's pretend we want to keep a leaderboard of the top five most liked tweets that contain the word tiger if you look at our tweet search answer key you'll see a bit about where this might come up 

so the sorted set commands all start with z and their syntax is quite simple I give a key the ranking value in this case the number of likes and then Su string identifier in this case I'm going to have the Tweet ID we'll call it Su ID 1 this first tweet let's assume it's a tweet about Tiger Woods in the next tweet we're going to do the same thing but for some ID 2 this might be a tweet about Zoo tigers and we're going to add the ID 1 remember I called these sorted sets which means for any given ID it can only have one ranking value so if for instance the Tiger Woods tweet got an additional like I would run the same command with 501 to update it now I can run this command Z range by rank to remove items that are in a specific rank or range of ranks and in this case what I'm doing is I'm removing all but the top five so what I effectively managed here is a heap I've got the top five most like tweets and every time I add a new tweet I can remove the ones that are not in there and I will only actually get an incremental uh example in this list when one of those new tweets Rises to a number of likes that is greater than my top five now it's important to remember that these run in typical sorted list times so as an example when I'm adding things to the list I basically am eating a log M where m is the number of items in the list uh complexity and so being able to run this command keeps my list small and manageable I wouldn't want to maintain an indefinitely growing list of the tweets that contain the word tiger because then every incremental addition needs to be inserted into that sorted list and while it's logarithmic it still is growing as the number of examples grows now this sounds like a pretty good idea but remember I'm doing this in a single key so in in so far as I need to manage uh the top like tweets by specific keywords I can probably distribute this across a cluster by having unique keys but if I was using this to keep track of all of the tweets and try to find the most like tweets across my entire uh setup then I might have a problem because they're all going to sit on a single node if I want them to sit on multiple nodes I'm going to need to keep multiple sorted sets and then combine them at the last minute so what I would basically do is take some hash of the Tweet ID and then I would keep a sorted set of the items that have that hash and then when I wanted to query this I would query all of those sorted sets across my cluster I would need to issue however many queries for the number of shards maybe 16 queries uh in practice this is usually not a big deal especially because redus happens to be so fast but it certainly does have limitations on on scaling and and how you build your systems once we have the sorted set primitive there's a lot of things that we can build on top and reddis goes out of their way to implement what I think is one of the most useful uh geospatial index the use cases for this vary but essentially if you have a big list of items that have locations and you wouldn't be able to search them by location this is a great way to do it and so the API looks pretty basic when I want to add an item to my geospatial index I'm going to pass in a longitude and in latitude and an identifier for my item and that's going to add it to the index at this key when I want to search that index I run the Geo Search Command give it the key I'm going to give it a an anchor point and then a radius and I can optionally return the distances that are associated with it and this works exactly like you'd expect if I have a bike rental service and I want to find all the nearby stations I add the stations to my GS spatial index and query the way that this works under the covers is that each of these longitudes and latitudes are geohash to give them basically a numeric identifier that numeric identifier is the ranking in the sorted set and then reddis under the covers is calculating the bounding boxes given your radius and then finding all of those entries that are basically within that range in your sorted set Rus does also in this case ensure that you're not returning items outside that radius so you theoretically wouldn't see an entry with a radius of six but the important thing here probably isn't the internals the important thing is that this API is super convenient and works in a wide variety of different situations now from a system design perspective this isn't without its perils there's a number of cases where you might not want to do this the first is if your items aren't changing location very very often if you're not updating your data it may actually be better to just keep a static list of longitudes and latitudes in the service that's actually making these queries and then just calculate the have sign distance for all of them if for instance I have a th000 stores across the globe that's actually not that much to do some simple uh floating Point arithmetic to go and find the closest place it's certainly faster than making a network call out to Rus the other instance where this can have some trouble is that remember that the index is on a single key which means it's on a single node if I need to go and separate this out and I need to go and Shard this then I've got to decide on some way to do that 

there's a lot of natural ways to do this I can calculate the geohash on my side I can take some of the most significant bits and use those as part of the key I can break this out by continent where frequently I'm not going to need a query across continents or or if I'm near the border maybe I'll query to maybe uh North America and South America but generally speaking I need to be concerned with how this scales if I've got say billions of examples the last capability of Redd that I'm going to talk about today is pubsub and pubsub is really trying to solve for this unique instance where your servers need to be able to address one another in a reasonable fashion canonical example of this would be a chat room where uh us user one is connected to server a maybe via a websocket or some persistent TLS connection and they need to message user 3 who unfortunately is connected to server C how does server a know user 3 is on server C there's a bunch of different potential solutions to this problem probably the most famous is to use a consistent hash ring so that way user 3 is always allocated to server C and server a knows that and so can send messages directly there but there's a bunch of incremental problems that happen with these consistent hash Rings notably it's harder to to manage the balance between servers and scaling them up and down requires a bunch of very careful orchestration usually with a service something like zookeeper but redus has this uh capability called pubsub and the idea with pubsub is that the servers can connect to redus and announce a a publication uh that they're going to be making and then on that topic other servers can subscribe so if for instance user one connects to server a server a is going to tell reddis that I have user one any messages sent to the topic for user one should come back to me and server C would do the same thing now when server a wants to send a message to server C so that user 3 gets it what it's it's going to do is publish to the topic of user 3 redis pubsub is at most once delivery so the idea here is that the message might get to a user and it might not which is surprisingly useful in spite of its reliability issues what it typically means from a system design perspective is if you need to guarantee that messages eventually arrive you're going to have to try something else but Reda pubsub is going to be very very fast it's operating on a single box and all it's doing is bouncing these requests between different services and so it can scale quite well and what this allows us to do is if we wanted to we could swap user one and user 3 they could connect to different hosts and redis pubsub would be the registry that knew which host each user was connected to this is something that's a little bit harder to pull off with for instance a consistent hash ring and it also means if server a were to go down we can migrate user 3 to server B quite quickly because server B will now register its publication to that topic and for a while those messages will go both to server a and server B but they'll eventually get their way to user 3 like I said there's a bunch of considerations with this I think the WhatsApp key that we have on the site talks about some of them in depth but be happy to answer questions in the comments uh if you've got them overall thank you so much for listening in on this deep dive on redus redus is useful across the board in a number of different instances from geospatial indexes leaderboards distributed locks and so it probably has a place in your system designs and I hope this presentation this video has really shown you that the internals of redus aren't that complicated and if you understand them then you can reason through some of the issues that you might expect in production so that's a wrap on our reddest Deep dive would love to get any feedback that you all have in the comments any questions or followups if you'd like to read more our redest Deep dive write up is available in the description below and we're looking forward to making more of these for you all if you find them interesting so let us know what you'd like to see next thanks I'm Stefan have a great day