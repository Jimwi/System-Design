hello streaming data is everywhere and its growth seems unstoppable 

activity from social networks website analytics and sensors and iot devices generates a constant flow of new data coming at us from a huge number of sources in this video we'll see how redis can help us capture manage and make sense of these large and constantly moving volumes of data right out of the box streaming data is data that's generated continuously often from a large number of concurrent sources one useful way of thinking about a stream is as a series of events every entry in the stream represents a new event for example you can imagine recording a stream of weather sensor readings at a given location every entry in the stream might consist of a temperature a humidity measurement and the wind direction and speed because streams and events are such a useful abstraction and occur so often in the real world redis provides a data structure called a stream 

in the rest of this video i'm going to introduce some streaming concepts and then i'll demonstrate some of the basic commands you can use to manipulate streams and redis 

so let's start with concepts 
in a distributed application architecture the components that write to a stream are called producers the data they generate is added to a stream with each entry having a unique identifier and its specific fields at the other end of the stream are one or more consumers these consumer processes read entries from the stream and process them as necessary what's interesting is that you can have more than one consumer reading from your stream each consumer has its own role 
for example one type of consumer might create notifications in a mobile application when certain trigger values are seen in the data another consumer might write all entries to a data warehouse for later analysis while a third consumer could act as a producer for yet another stream adding only a subset of the entries to the new stream producers and 

consumers often operate at different rates the stream acts as a buffer between them as well as the decoupling mechanism this means that producers and consumers don't directly communicate with each other so don't need to know anything about each other's implementations 

now let's talk about the reddit stream data structure itself a reddit stream is a data structure that behaves like an appendally log once added an entry in a stream is immutable each entry in a reddest stream has a unique id and by default these ids are timestamp prefixed this means that a reddest stream keeps entries ordered as a time series stream entries look a lot like a redis hash each is a set of name value pairs it's also worth noting that reader streams are schema-less while each stream entry has to have at least one name value pair they don't all have to use the same structure redis allows consumers to read stream entries in order consumers can also efficiently seek to any entry within the stream so enough theory let's look at redis streams in action we'll see how they fit into a common use case for streaming data systems recording real-time crowdsourced data 

imagine we're building a mobile app that allows users to check in at all sorts of businesses public spaces and workplaces. 

users provide star ratings based on their experience at each location 

each time a user checks in they'll pick out their location from a list our app provides 

they'll then select a star rating from 0 to 5 and the app will send this data along with their user id to a server 

users earn prize draw entries for each check-in and we periodically offer cool prizes to randomly drawn winners 
this incentive encourages users to check in often improving their chance of winning


a typical check-in can be represented as a set of name-value pairs 
here we have a check in for user id 99 at venue id 103 the user decided to give this venue 4 stars 
 
readings from our many users are sent continuously to our servers and those servers produce entries into our redis stream 

once this fire hose of jumbled data is added to a reddit stream we can begin to organize and make sense of it 

let's see how each check-in finds its way into a reddit stream 

first like other redis data structures a stream is associated with a redis key 

we'll use the key check-ins to identify the incoming stream of check-in data the x-add command adds a new entry to a stream 

here we're adding an entry to the check in stream from user 90781 this user is visiting location 348 and rating it three stars the asterisk in the exact command tells redis to assign this entry a unique id consisting of the current timestamp plus a sequence number 

exside returns the id that redis is assigned to the new entry 

the first part is the timestamp in milliseconds and the second is a sequence number 
since stream ids must be unique. 
this convention ensures that we can add as many entries as we need in the same millisecond 

now that we have check-ins flooding into our stream it's time to think about how the business can make sense of them 

as redis assigns each entry a timestamp based id one way that a consumer can view the stream is as a time series 
here we're using the x range command to read entries in the check-in stream that fall between the specified start and end timestamps 
x-range returns each entry whose id falls within the specified time period. 
entries are returned in order with the oldest first 
to limit the number of entries returned we can use the optional count modifier 

if we want the most recent entries first we can use the xrevrange command instead 
note that here we specify the time period in the reverse order with the end timestamp coming first and 


if we don't know what time period the entries in the stream cover we can use both x range and xrevrange with a special plus and minus operators to represent the highest and lowest timestamps respectively here we're retrieving the oldest two entries in the stream 

but really streams are all about real-time data consumption so we want to build consumers that continuously receive data 
we could achieve this by pulling the stream using the x-range command but that's inefficient 
ideally we want a command that lets us consume the stream blocking when we've seen all the entries until a producer adds new ones 

this is one of the use cases for the x read command 
xreed can consume one or more streams optionally blocking until new entries appear 

here i'm calling x-read against the check-in stream 
x-read consumes the stream returning entries whose ids are greater than the one provided 
here i'm asking for all entries with an id greater than 0, the beginning of the stream.
and redis returns the entire stream 
 
we'll need to note the id of the latest entry for subsequent calls xread. 

we can also invoke x-read in a way that blocks the consumer until new entries are added to the stream 

to use xread in a blocking context i'll provide the last entry id for my previous call 

i'll also specify i want to consume a single new entry and how long to block in case the stream doesn't yet contain any entries newer than the one whose id i'm supplying 

here i'm telling x-read to block for up to 5000 milliseconds or 5 seconds when i run this command it blocks because no new messages with higher ids than the one i provided have yet been added to the stream 

while the consumer is blocked i'll use a second redis cli session to add a new entry to the stream as soon as i do that the consumer unblocks and xreed returns the newly added entry if no new entries have been added in a 5 second blocking period xreed would have returned a nil response 
i could then choose the block again or give up trying and do something else 

it's important to note that consuming the stream using xrange  xrevrange and xread doesn't remove entries as they are retrieved 
entries remain in the stream allowing other consumers to read the entire data set, each processing it in its own way 

here one consumer is maintaining running average star ratings 
a second writes the entries to a data warehouse 
and a third adds entries to another stream if the star rating is below a triggered threshold 

so if new check-ins are constantly pouring into our stream and consumers reading them aren't removing them then how do we contain this seemingly never-ending growth 

remember that a stream models independently log, this means that the order and content of entries can't be modified once they've been added 
however redis supports a trimming strategy to manage a stream's memory use 
let's assume our stream now contains tens of thousands of entries and that we need to control its growth 

trimming a stream removes its oldest entries that is those of the lowest timestamp ids.
this frees up memory associated with these entries 

let's see how that works in practice 

redis commands can be used to trim a stream 
the first of these is xtrim 
this command can be run at any time 
and trims the stream's length to a specified new length 

here we're using xtrim to trim our checkins stream to a new maximum length of 10 000 entries 
the command returns the number of entries removed 
in this case we removed 20 000 in one of the oldest entries from the stream leaving the newest ten thousand 

the second way to trim the oldest entries from the stream is to do so while adding a new entry with the exact command and this is the more common strategy if we know that we always want to keep the checkins stream to a length of 10 000 or less 

then we simply specify that on every call to xadd, like you see here. 

as you saw earlier xad returns the id of the entry that was added but doesn't tell us what the new length of the stream is for that we can use xlan and we see that the stream has been capped to 10 000 entries including the newly added one there's a lot more to reddit streams of what i've presented here so if you're interested in learning more you should check out redis streams our free online course from redis university i know streams can seem like a fire hose of information so thanks for sticking with me and forwarding this river together happy learning and see you again soon


neuendorfer strasse 52
