hi everyone welcome back to the channel.

Today we're going to do a deep dive on kafka and this is going to be specifically through the lens of a system design interview 
so at a high level, kafka is a event streaming platform and it can be used either as a message queue or as a stream processing system.


Now why does CFA matter at all, well according to their website it's used by 80% of the Fortune 100. 
you can actually see some of those logos on your screen now. 
these are all companies that use Kafka 
and according to us at hello interview it's one of the top five technologies that we think you should familiarize yourself with in order to Ace those upcoming system design interviews 

Now my goal here is to give you just enough information to be comfortable knowing when to use CFA in your interviews and to have enough depth importantly to be able to speak in detail about some of the trade-offs necessary to impress your interviewer.

so the way that we're going to do this is we're going to go through four sections 
I'm going to lead with a motivating example uh then we're going to give an overview on Kafka it be pretty high level 

I'll talk about when you should use it in an interview and then importantly everything that you need to know in order to show that depth in an interview context 

Now quickly for those who may be new to the channel I'm Evan I'm a former meta staff engineer and interviewer 
I've conducted many hundreds of interviews and nowadays I'm the co-founder of hello interview which is a site that helps software Engineers prepare for their upcoming interviews 
via free educational resources as well as paid mocks with current or former Fang interviewers 

and so lastly if videos aren't your thing I've linked a written guide in the description as well uh 
and this is where I go into all the same topics I discussed in the video just in written form so you can head over and check that out if that's something that you prefer uh 

you'll see all of the other great written content that we have there on the site as well 
so without further Ado let's jump into it 


####
When we talk about Kafka I find it useful to start with a motivating example and I'm going to lean on what is my favorite sport and my favorite competition in the world cup  

so shout out to everybody else that might be listening that similarly loves soccer or football 

but for this example we can imagine that we're running a website that's going to provide realtime event updates on each of the games.


so every time a goal is scored or a player is booked, a substitution is made.  we're going to update our website with the latest information. 

now the way that this works is that events when they occur are placed on a queue 

We call the server or the process that's responsible for placing these events on the Queue the producer. that terminology is important 

the producer might just be somebody with a laptop who's sitting at each of the games. 

Now on the other end we have a server that reads Events off of the queue and updates our website. we call this guy our consumer 


Now our website is working great through the first handful of games 
our events are being processed in order, they're updating our website and users are really loving our product.

But all of a sudden FIFA throws a wrench in our project.

They decided that they want to expand our hypothetical World Cup from 48 teams to a thousand teams 
And they're also going to start all of the games at the exact same time 

So now the number of events has increased significantly and this single server here that we have that's processing or responsible for holding our events in the queue is really struggling to keep up 
We need to figure out how we can scale this. 

And so with too many events on the Queue that server is running out of space 
and the solution is one that you've probably read plenty about as you've been studying for system design uh 

And that's to scale horizontally or to put it more simply we're just going to add more servers 
and so in this diagram I have two additional servers that's great 

You could imagine that we have three four five six it can scale largely indefinitely um 

But we have a new problem and that's that as a producer produces an event if we're going to randomly distribute these events across the servers, we'd end up with a mess on our hands 

goals are going to be scored before the match even started. players will be booked for files that they never committed 

and this is because the consumer is just reading off of both of these queues 

and there's no way for us to maintain the Order anymore. you can see this example if it read the sub in the 83rd minute before the goal in the 80th minute, then our events would be out of date. 

and so now that new problem events are not processed in the correct order 

and so the solution to this is that we can distribute the items in our queque based on the game that they're associated with 

so for example this top Queue is going to have everything from that Brazil Argentina game and this Q down here will have everything from the France game 

this way all those events from a single game. we can be sure that they're processed in order because they exist on the same queue. 

so we can't guarantee that this 81st minute goal from France happens after the 80th minute goal from Argentina. but that's okay. because what we really care about is that this event happens before this event 


and this right here is one of the fundamental ideas behind Kafka and it's that in order to scale messages sent and received through Kafka require a user specified distribution strategy 

uh basically the user needs to specify how we distribute or partition events across our different partitions 

so this is working well but we've run into a third problem 

and the third problem is that we've scaled to so many events or so many queues here 
and our producers are producing so many events that our consumer can no longer keep up 

the consumer feels like it's uh what's the saying it's it has its mouth open at the end of a fire hose it can't drink fast enough and 

so we have another logical solution there which is that we're going to add more consumers 

but if we were to just add more consumers then the problem is consumer one might read the goal from Argentina and consumer 2 might read that same goal from Argentina maybe even they do it around the same time 
and then our website ends up with two of the same events 
and our users think that Argentina scored two goals which can't possibly have been the case we just messed up 

and so the solution is something that Kafka calls consumer groups and with a consumer group each event is guaranteed to only be processed by one consumer in the group 

so we can take these three consumers we can call these three individual servers either physical or virtual we can group them into a Kafka consumer group and this makes sure that each of these events is only going to be processed by one of these consumers in the group 

Now the last twist that I'm going to throw at you before we're done with this motivating example is that all of a sudden FIFA decided that they want to expand this hypothetical World Cup to even more Sports something like basketball uh but we don't want our soccer website to cover basketball events and we don't want our basketball website to cover soccer events so we introduce A New Concept that's called topics 

so let's look at that here 
now each event is going to be associated with a topic and consumers will subscribe to a specific topic 

so therefore our consumers who update the soccer website only will consume from our soccer topic and the consumers that update our basketball website will only consume from our basketball topic 

so our consumer groups can specify the topic that they consume from and likewise our producers when they produce something need to specify the topic that they want to write to 
and then within the topic we can have multiple servers and multiple partitions 

so what we've described here is basically the linear reasoning that gets you to Kafka and a Kafka cluster 
and so from here we're going to break down uh or kind of move away from this abstract motivating example and talk about actual Kafka and the terminology that Kafka uses in a bit more detail without this layer of abstraction.

all right so as we went through that motivating example we introduced a bunch of key terms I want to take a moment to introduce a couple more and really just kind of solidify a few of the ones that we mentioned via the example 

so the first term here is a broker and a broker is simply a server 
it can be physical or virtual 
but Kafka clusters are made up of these Brokers, these sets of servers 
and what these servers do, what these Brokers do is that they hold those queue. so they're responsible for holding the queues here 

Now the queues themselves are actually called partitions in Kafka and what they are is an ordered immutable(that's important they can't change ) sequence of messages that we append to 

you can think of them just like a log file it's a log file that exists on disk that we append messages to 
and so each broker each server can have multiple of these partitions that's totally fine 

and then the next concept that we talked about briefly were these topics and so importantly these are just logical groupings of partitions 

and so when you publish a message you publish it to a topic 
when you consume a message you consume it from a topic 

um a question that I get often and that confuses folks is what's the difference between a topic and a partition and so I want to make this as clear as possible and that's that a topic is just a logical grouping of messages it's a grouping that exists in code where a partition is a physical grouping they're actually the separate log files that exist on disk and so a topic can have multiple partitions and topics are just ways of organizing your data while partitions are ways of scaling your data so hopefully that makes the distinction between these two relatively clear 

and then last up the easy ones producers and consumers. 
uh producers of course write the messages or records to topics. consumers read them 

we have our producers and our consumers here hopefully that part was relatively clear 

So let's zoom in a little bit now and let's look under the hood let's get rid of the abstract and let's talk about exactly what happens in a life cycle of a message uh through Kafka through a Kafka cluster and so the very first thing that happens is a producer is going to create or publish a message well what is a message first off I'm going to use the term message because message cues we're familiar with with that terminology and I want to remain consistent regardless the technology but CFA for what it's worth actually calls these records 
so we might use these words interchangeably.
but a message or a record in Kafka is made up of four attributes.
importantly we have the key and we have the value 
and then we have a timestamp this is what determines the ordering if none is specified then we just use the machine time and then we have some headers these are just like HTTP headers the key value pairs that you can specify some additional information. 


So how does this work well you can actually go download cfco on your local machine if you want to right now and you can download uh this CLI for it and so if you want to interface with CFA via CLI you'd run a command like this a CFA console producer is the command we're going to write to the topic my topic and then these additional property Fields just talk about how you should split up our string inputs here into an actual message so don't worry too much about them they just say that you should parse or separate on colon and what this is going to do is it's going to add two messages the First with a key of one or key1 excuse me and a value of hello cfco with key and the second key is key2 and another message with a different key so now both of those message messages are on our queue uh looking quickly if you don't want to use the CLI of course you can use any of the common libraries that exist for all of the popular languages this is Kafka JS for JavaScript and you can see how simple it is you just create a producer and then you write producer. send and you send it to a topic and a list of messages that you want to send to the Kafka.

so the second thing that happens then is that once the Kafka cluster receives that message, it needs to determine where it should go put it 

so it has to determine where it should put it 
so it has to determine what the topic is, what the broker and what the partition is. 

and the way that that works is that it goes through this flow 

when it gets the message it first determines does that message have a key and so the key also known as the partition key is What determines which partition the data should go on in our motivating example this partition key was like the ID or the name of the game Argentina versus Brazil for example 

now if you don't specify a key which is okay keys are optional then we're just going to round robin or randomly assign this message to one of the partitions this works for simple cases but not if you really need to scale but that's this path 

now the common path is that you did specify a key so you have a way that you want this to be partitioned and so what we do is that we take that key we hash it. this is actually using I believe a murmur hash which is just a a fast hash function and then we take the modulo of that hash over the number of partitions that we have here say n 
and this is going to give us a number and that number is going to correspond to the partition 

so now for example we know that Argentina versus Brazil should go to partition five and of course uh this is deterministic.


So the next event that comes in here with a key of Argentina versus Brazil will also need to go to partition five in this hash function plus the modulo will ensure that it does 

and so now we have the partition that it needs to go to. 
but the first thing that we need to do is we need to identify what broker or what server does that partition exist on 
and so there's a controller that exists in the CFA cluster that has some mapping it keeps the mapping uh of partitions to Brokers and we're going to look that up. it's like a simple hashmap and it's going to tell us which broker we want to send it to 

we're going to send it to that broker. that broker is going to receive it and append it to that append Only log file 

and so everything that happens here is kind of uh behind the scenes. it happens native to the CFA cluster 

this is native to CFA code you can read it it's open source but you don't really need to worry about this. all that you worry about is setting the appropriate key in order to determine how we partition 

so when the broker appends the message to the correct partition what happens is that it just appends that message to that append Only log file and that log file looks something like this you have message A B C all the way through e and we just keep adding messages and each of these have a corresponding offset 0 1 2 3 Etc and so now as consumers want to consume a message they consume a message by specifying the offset of the last message that they read so if they already read message a they know that they've consumed message a with offset zero same too with message B you know if they've already read message B and message a then they know their latest offset was one and they're going to ask then for offset two the next time that they want to go uh read from the queue um 

now what happens if for example that produc or that consumer excuse me went down and it no longer knows its own offset it doesn't know the last offset that it read 
well the way that this works is that it's going to periodically commit its offsets actually to Kafka 
and this is important because now Kafka maintains what the latest offset was that was read 
and in the case where we have a consumer group like we had above then consumer 2 can read uh and ask Kafka for the next message and Kafka knows what offsets have already been read by the other consumers in the consumer group and it can give you the latest message and then commit uh that offset so everybody else knows uh you know what the latest offset that has been read is so we read the message from Kafka we get that message back we maintain the current offset locally in the consumer periodically we commit those offsets and then in case of failure or restarts we can just resume reading from exactly where we left off because we know the offset that we last read from similar to the apis for producers just a quick look here if you want to consume using the CLI then you have a CFA console consume commands here uh we want to consume from topic my topic we can say that we want to consume from the beginning we could also consume from a specific offset and of course that'll return to us the two messages that we added earlier now you also have the CFA JS library look at what happens here importantly we subscribe that's the terminology we subscribe to a topic so this consumer subscribed to topic my topic and then for each new message that comes in on that topic in this case we're just going to print them out and so similarly we're going to just print out those messages and we'll get those there 

so let's tie it all together now and just quickly the last concept that I'll only briefly introduce is that in order to ensure durability and availability we have a robust replication mechanism in CFA and so each partition is designed a leader replica like this one here and it resigns on one of the Brokers and that leader replica is responsible for handling all the read and write requests to the partition but the assignment of the leader replica is managed centrally by some cluster controller which ensures that each partition's leader replica is effectively distributed across the cluster to balance the load 

and then importantly we have those followers and so for example we got a follower for partition one right here 
uh and these can reside on the same or different Brokers but these followers don't handle direct client requests instead they just passively replicate the data from the leader 
and by replicating the messages received by the leader these followers are just acting like a backup 
and they're ready to take over if the leader happens to go down 

so let's look at everything that we have here. we have our producers The Producers interface with our Kafka cluster via that producer API that we looked at 

Our Kafka cluster is made up of Brokers which are just servers and our case we have two of them

Existing on these servers are a bunch of partitions which are just append only log files 

and then these partitions are grouped logically or labeled based on a topic and so in this case we have topic a and topic B 

and then consumers consume via that consumer API and they consume a specific topic and so a consumer is going to subscribe to say topic B and it's going to read all of the messages off of topic B 

Again as I mentioned earlier consumers can be grouped into consumer groups and this just makes sure that the consumers in the consumer group don't uh read the same message only once so they don't read the same message twice 
any message is read only once 


###
When to use kafka

Now let's talk about when we would use Kafka in an interview so the first time is any time that you would need a message CU well what could those times look like let's look at a couple examples so the first is if processing can be done asynchronously in a key example of that is with YouTube so with YouTube when you upload a new video you upload that full video and you store that video in S3 but then you need to transcode that video basically take that video and put it into 480p 720p 1080p Etc so that it can be streamed at different qualities on all different devices but this transcoding process takes a while and it's going to happen asynchronously the user doesn't need a synchronous response 

so what we do is that we put these messages in CFA this message could look something like like you know video ID 123 and a link to the videos, uh the actual video in S3 
and then our consumer is our transcoders so they pull those messages off they download the video from S3 they do all their expensive transcoding and they store it back in S3. 
so because this processing could be done asynchronously Kafka was a great buffer in between there 

another is when we need in order message processing. so you can imagine in an event booking service like Ticket Master for really popular event we might want to put people under a waiting queue and only let people off of this waiting queue in groups of say a 100 or so every so often and this is to make sure that there's not a bunch of contention all at once for the same seats 
so what we can do is that when a user tries to view that popular event we could put them on Kafka on our waiting queue so our event service here is our producer and then our event service is also our consumer so periodically maybe every five minutes in the simplest case one minute uh we can pull the next 100 off of this que and then let our clients know that it's their turn uh to actually start to book this event 

now of course these diagrams are oversimplifications. on our website we have detailed breakdowns of each of these problems that I'm mentioning that you should check out if you want more details on these problems specifically 

uh the other one here as it pertains to message cues is when you want to decouple the producer and the consumer and the reason that you typically want to do that is because you want to scale them independently you want to horizontally scale them independently 

and so in this this case you can imagine that if we're running a competition for a site like Leetcode where users submit their code via competition we need to run their code and then give them the response 

um what could happen is that a bunch of users could submit their submission all at once maybe 100,000 users or something like this 

and so we can scale this primary server accordingly we can have a ton of these horizontally scaled boom boom boom boom boom right whatever you get the picture 

um and then they're just going to put it on CFA and the reason for this is because we don't want to scale these guys uh you know these servers that are actually doing the running of the code the containers U maybe because we're cost sensitive and so we can scale just this one put them on a CFA Q, the worker is going to pull it off the que and then give it to one of these guys to run or maybe these guys are just the workers themselves so we don't need the intermediary 
but in this case we could scale the primary server in order to handle that burst and there can only be one or two each of these uh so they don't have to scale 

Now the other popular use case not a message CU but very similarly would be a stream and so a stream is used uh when you need to process a lot of data in real time 
you can imagine that with the queue the consumer gets to choose when it wants to read off of the queue and things can exist in the que for a long time 
in the case of a stream you have just kind of an infinite stream of data that's being produced onto CFA and you need to be consuming it largely as quick as possible in order to give your users some realtime statistics or updates 

and so one common example is a problem again that's broken down on our website called ad click aggregator and so when a user clicks an add we put that click saying they click that Nike ad on our CFA que.
and then we have some consumer in this case it's Flink that's going to be reading off of this stream in real time and aggregating those so that we can tell our advertisers how many times their ad has been clicked 


and the other one here is that if we want a stream of messages to be processed by multiple consumers simultaneously this is also known as Pub sub and so what can happen here useful for like things like messenger and FB live again two that are broken down on our website is that if we have a commenter who leaves a comment on a live video then we can put that comment on Kafka as a pub sub and then we're going to have all of these other services servers excuse me that are connected still to the users that are watching that live video and they're subscribed to this stream and if they see a new comment coming on a live video that they know that their user their client is subscribed to then they're going to read it and they're going to send that user that new comment so that it looks like that comment showed up in real time to that user so that's another example these Pub Subs uh for which this is useful 


Now we're going to spend some time talking about what you should know about CFA for deep Dives in interviews if you've read any of the content on hello interview or if you've watched any of our YouTube videos then you know our recommended framework ends with first documenting a highle design of the system and then using that highle design in order to expand upon it by going deep in a handful of areas and this is where you show off that technical depth or that technical excellence and so if you introduce Kafka into your system it may be a place where you want to show some of that technical depth and there are really four areas that it typically makes sense to do this of course this list isn't exhaustive but I found in the many hundreds of interviews that I've done that that this kind of makes up the majority of the follow-up questions you may see and 

so the first is scalability how do you scale the system and consequentially how do you scale Kafka 
the second is Fault tolerance and durability 
the third is errors and retries what happens when things go wrong 
for four is performance optimizations so this is for specifically those realtime use cases when you have a lot of events how do you make sure that your throughput is as high as possible 
then the fifth is retention policies this is particularly to minimize the amount of storage that's persisted on disk 

so those are the five things that we're going to go into 

and we're going to start with scalability and so scalability is by far the most common follow-up question that you're going to get in a system design interview as it pertains to Kafka it's that question of how is this going to scale usually that's in the context of the full system 
and Kafka being a part of your system will require some attention there 

so before we talk about how it scales there are some constraints that are worth knowing and so the first one here is that actually there's no limit on the maximum size of a message in Kafka Beyond of course the natural Hardware limits of course 

but it is highly advised for you to keep those messages under 1 Megabyte 

this is so that you can ensure Optimal Performance otherwise you're going to overwhelm Network or memory 

and a common mistake that I see candidates make all the time is they put the entire media blob on Kafka so let's let's draw out an example here if you remember that YouTube example that we discussed earlier you have some Kafka q and it's responsible for uh storing videos or at least pointers to videos as we'll see in a second that then a transcoder worker is going to use in order to transcode them into different resolutions so this works on different devices right so this is the general flow and so in the Bad Case what you're going to do is you're going to have a message where the key is something like video ID and then the value is the video blob itself and that video blob of course is huge that could be up to a gigabyte and so that's a huge message here on CFA and so this clearly is bad and something that I see all the time now the good alternative over here is that instead what you do is that you minimize the size of the message on CFA and maybe you make this just an S3 URL and so now you have S3 here when the uh producer was going to put something on Kafka it also writ that wrote that video to S3 maybe that happened earlier in your system as well ideally and then what your transcoder is going to do is it's going to go up to S3 download that video and then do the transcoding and it's going to download it just via that S3 URL and so now the the only thing on Kafka is the video ID small 16 bytes and an S3 URL also pretty small so this is a common anti pattern that I see candidates to all the time and is something that you definitely want to avoid now the other constraint worth mentioning is that uh one broker and this is for kind of good Hardware highly optimized Hardware a single broker can store about a terabyte of data so a terabyte worth of messages and can handle about 10,000 messages per second so no this is very handwavy uh it totally depends on the hardware it totally depends on the size of the message but in an interview if you're doing back of the envelope estimations these are kind of good baselines for you to use here and then in your interview you should do the math you should see does your system exceed this um and then if it doesn't maybe you don't need to worry about scaling it all so easy say that to your interviewer it's fine I'm gonna have a single broker here no problem at all we can handle this scale now uh in the case that you do need to scale what should you do so if you need to scale then there's two steps the first is of course you're going to introduce more Brokers you're going to introduce more servers more Brokers means more memory means more disk which means that you can store and process more messages so that's step one step two as we've discussed and it's the most important step is that you need to choose your partition key so this is something that you're likely going to spend a lot of time talking about in the interview and you need to decide how you're going to partition this data across the Brokers and this is done by choosing a key for your message messages so if you choose a bad key then you can end up with a hot partition or all of your data showing up on a single partition that's overwhelmed with traffic um and other partitions that get no data good keys of course are ones that evenly distribute across the partition space now I want to talk about handling hot partitions but actually before I do that let me just mention that it's worth noting that nowadays many of these scaling considerations Are Made Easy via managed cfus Services you have like uh confluent Cloud AWS msk are popular ones and these are managed Services hosted by Cloud providers that handle kind of all of these scaling things for you for the most part you're still going to need to pick your own partition key but everything else is done largely for you will automatically scale up Brokers for example so that's good to know you can even mention it in your interview that you're going to use a managed instance but it's still really important for you to understand the underlying Concepts okay so back to that that hot partition how do we handle hot partitions again a hot partition is that you can imagine that you've created a partition key now uh for which everything is going to a single partition a single log file and a single broker and it's overwhelmed right so um I think a good example here um is to consider an adclick aggregator so an ad click aggregator in that question which is a really common question what you have is a stream of data where that data is that user a clicked on ADD B right and then we take that stream of clicks and we aggregate them so that advertisers can understand the metrics of how often their ads are being clicked on naturally what you would do is you would Partition by ad ID but then let's say that Nike launches a new LeBron James ad then you better believe that that partition for that Nike ad is now overwhelmed with traffic and you're going to have a h hot partition on your hands and so how do we handle it here those three steps that we can do the first and most obvious but maybe the easy way out is that you can always just remove the key from your messages in doing so you're going to just randomly evenly distribute your messages across partitions in the case where you don't care about ordering at all this is fine this would work sometimes it seems too simple but if you don't need ordering this is a great option now second um is that you can and this is the most common that you can add a compound key so where our key was ad ID we can make our ad ID colon and then something else and so really common thing to do is let's say that we want to take our Nike ad and we want it to be partitioned across instead of going to a single partition 10 more partitions so we can add a random number here between one and 10 and this is now our key add ID colon random number 1 through 10 and this is going to make sure that now the message for these Nike ads are split across 10 different partitions and thus reducing kind of that uh you know that strain on that single hot partition another thing that you can do is you can append another value to it say the user ID or even you know the prefix of some user ID if this is sequential um but these compound keys are effective for just taking a single hot partition and then spreading it out now of course you're going to have to have some logic in the producer that understands which partitions are hot and then does this concatenation for just the keys on those partitions you also then have consequences to the consumers in that of course now there's no ordering across for example these 10 these are all things you have to keep in mind and then the other option here is back pressure and so this is pretty simple our producer can just understand that a given partition a given topic is overwhelmed at the moment and we're going to slow down our rate of production this does not work for kind of a large number of services but for some it's a reasonable thing to consider so next up we're going to talk about fault tolerance and durability one of the reasons that you might have chose cafin your interview is because it has really strong guarantees on durability and we talked a bit about how it does that so I'm not going to go into too much more detail there if you remember we have a leader partition and then there's a number of follower partitions which are there to take over in case the leader ever goes down now the things that are more important here is that there's two relevant settings this is kind of like the how would you actually do this and so when you set up a CFA cluster you have a config file and two of the settings are AXS and replication Factor ax is just saying how long do we have to wait or how many followers excuse me need to acknowledge that they've gotten a message before we can keep going before we can keep moving and so a all is maximum durability this means that every single follower needs to acknowledge that they also got the message and then we can say that we got this message we'll take the next message now uh the trade-off here is of course durability versus performance if you have your ax value set to fewer then we're only going to wait for at least you know for example one follower to acknowledge it and then we can move quicker but it may be that the other ones didn't actually get it in time in case something goes down hopefully that makes sense the second is that replication Factor this is just how many followers you're going to have this is also set in the config and the default is three if you want more durability you can increase this number if you want to maximize uh you know or or take advantage of limited space for example then you could make this number smaller but just know that you're going to be less durable durable so that's something you could talk about in the interview um now what happens when a consumer goes down let me go over here the first thing that I'll say is that Kafka is usually thought of as always available so if you ever get a question from your interviewer like what happens if CFA goes down uh this actually isn't very realistic it's not a great question you may even want to gently push back on the interviewer if they ask this like KF doesn't go down and the reason that it doesn't go down is because the durability guarantees that we just discussed a more realistic question is this what happens when a consumer goes down and so as you remember if we have the CFA cluster here our consumer first reads a message and then it gets that offset of the message that it just read and it comit commits it back to Kafka excuse me tripping over my words a little bit it commits it back to Kafka so if it just read message 100 it tells kafa I read message 100 and then if it were to go down it's going to come back up and it's going to pick back up where it left off at message 100 if you have a consumer group so maybe you have three of these here and one of them goes down then they're all responsible for different partition ranges is the way that it works so if one of them goes down then you need to have a rebalancing and these two need to get different ranges um of partitions that they're going to need to read from and that's all going to be handled by the Kafka cluster it's going to update the consumer on the new ranges the the last thing that I'll mention here is that it's really important when you decide to commit your offset so committing the offset is you saying the consumer has finished the task that it needed to do so in the case of WebCrawler for example we don't commit our offset until we know and have gotten confirmation that the HTML that we downloaded from the website we're scraping has been stored in S3 at that point our job is confirmed to be done and we can commit the offset the reason that we wouldn't commit the offset before we know that the uh data was stored in S3 is because then if our consumer went down let's say that our consumer pulled the message immediately committed the offset and then tried to parse uh pull the data from the internet internet and store it in S3 again if our consumer then went down then now we've committed our offset already and we're going to think that we already processed that page when we didn't so when you commit the offset is important and it should be once you're done with any logical set of work this is why you really want to keep your consumer the amount of work that they do here as small as possible now let's talk about errors and retries so first it's important to note that cfet self- handles most of the reliability as we saw above um but our system May Fail getting messages into or out of CFA so that's why we need to handle these scenarios gracefully either producer retries or consumer retries specifically so on the producer side it's pretty straightforward we might fail to get a message into the CAF cluster due to network issues or broker unavailability or any of these things and to handle this uh CFA producer API supports a couple of configurations that allow us to retry gracefully so in this case we've enabled up to five retries and we wait 100 milliseconds in between each of those tries there's additional Keys here or Fields configurations that can be used for like exponential retries but the general concept is simple here just keep trying until you get an acknowledgement from the CFA broker that we've received this message now importantly you'll want to ensure that you enable item potent producer mode and this is to avoid duplicate messages when retries are enabled so this just ensures that messages are only sent Once In the case we incorrectly think that they weren't sent so you want to make sure that that guy's there now what's more interesting is these consumer retries so let's paint an example here we'll use WebCrawler again this is our our CFA Q or our Kafka broker and it's storing URLs of websites that we need to crawl our consumer pulls these URLs and then goes and fetches them from the internet in order to get that HTML and store it off into some blob storage like S3 now the website's a dirty place and so maybe this main topic which has all of the URLs that we need to fetch we pull them we go try to get it from the internet and that sites down so Kafka actually does not support consumer retries out of the box AWS sqs uh kind of an alternate Cloud message cue does and that's interesting in your system design interviews if you need consumer retries you may want to consider sqs but the common pattern that's used and you can do it yourself is that you pull off that main topic you do whatever you need to do and if it fails then you're going to place it on a retry topic after the first end failures so maybe I put a new message on a new topic for retries specifically where that message says the number of retries and then either the same consumer or a different consumer can be pulling off of that retry topic and for end times just adding it back to that topic for us to retry now eventually if that n the number of retries exceeds some value that we set say five then we know that it's failed and we can put it on what's called a dead letter q and so this can just be another topic we call our dlq topic and no consumer reads off of this it just sits there largely indefinitely Or until it's purged so that some engineer or somebody on the team can go look at these ones that failed if they want to and try to do some debuging so that's how we're going to handle producer and consumer retries respectively now especially when we're using cfas a stream to process realtime data we need to be mindful of the performance and the throughput so that we can process messages as quickly as possible and so two things that we have at our disposal to do this both on the producer side the first is that we can batch messages in the producer so this is that we have fewer requests and this again is just a configuration on the producer so we can set you know the maximum batch size in bytes and the maximum time to wait before sending a batch and so what this is going to do is it's going to just aggregate a bunch of messages within this time period or until this size is reached and then it's going to send a single request to the ctica broker with all of those messages and then the second thing that we can do is we can compress messages in the producer and so this compression can happen for you via this producer SDK here as well at least with CFA JS and you can just specify for example gzip that's your compression algorithm and so each message is going to be compressed which makes them smaller which means that we have fewer bytes to send over the wire so both of these things by batching messages that have fewer requests and by compressing them to have smaller data we're increasing throughput now importantly arguably the biggest impact you can have on performance comes comes back as it always does to the choice of that partition key so the goal is to maximize parallelism by ensuring that messages are evenly distributed across all of those partitions and across all those Brokers so in you're interview discussing the partitioning strategy or the key that you're going to use as we go into above should always be the first place that you start the last thing that we'll touch on briefly is the retention policy very simply this is how long are we going to keep messages around before we purge them and so Kafka topics this is configured per topic have a retention policy that determines this it determines how long messages are retained in those logs and this is configured via two settings the first is retention. Ms this determines how long we should keep the message in milliseconds and the default if you don't change anything is seven days and the second one is retention. bytes so this is basically saying at what size when the log gets to how many bytes should we start to purge older messages the default here is 1 gabes and so it's whichever of these comes first if the log becomes 1 gab then we start to purge if 7 days have passed then we'll delete those older messages and so in your interview you may be asked to design a system that needs to store messages for a longer period of time than this you can imagine that you need to replay events for some reason even a month two months later uh in this case you can configure the retention policy to keep those messages for a longer duration you'll just want to be mindful and call out in your interview the impact that that's going to have to storage costs and to Performance all right great job everyone everyone and congratulations if you made it this far hopefully you found this informative uh and you should be well equipped now to be able to know when to use CFA in those upcoming interviews as well as respond to any follow-ups from your interviewer or lead any deep Dives uh we're going to continue posting both written content and videos so keep an eye out for future deep Dives we've posted Reedus already of course we just went over CFA we'll have more in the future as well as the breakdowns to some of the commonly asked system design questions we've also started posting some interviews with other folks so there was an interview between Stefan my co-founder uh and Christian who's an em or an engineering manager at meta you should go check that out if you haven't seen it already super informative two really smart guys um now if you have any questions or you want to point out anything that I might have gotten wrong or just say hi feel free to leave a comment below I try to get those get to those as quickly as possible um so feel free to to say what's up there and then the last thing is that if you want to practice with me or somebody like me head over to hello interview.com and consider scheduling a mock interview so of course I'm bias um but I get messages from candidates every single day where they're effusive about how helpful the mock interviews were in their preparation and how would help them eventually land a role at one of these major tech companies and so I'm fully bought in that these are kind of some of the most effective means of preparation possible um and so if it's something that's interesting to you head over there check it out we'd love to get a chance to work with you all right uh thanks so much and as always good luck with those upcoming interviews