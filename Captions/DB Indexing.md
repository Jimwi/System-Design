today we're going to do a really quick high level overview on database indexing and so this is going to include talking about uh what is an index? what problem does an index actually solve? what are the most common indexes and when should you use them?

Now as I said it'll be high level and for much more detail please head over to the hello interview website and we have a deep dive on database indexing I'll be sure to link that in the description let's get after it 


All right so before we start talking about indexing let's understand the problem that indexing solves so data is arranged in a database in Pages pages are usually 8 kilobytes of data and so when you want to find a particular item in your database and you're not using indexing what happens is that we pull a page into RAM into memory we scan through all roughly a 100 rows or items looking for the item that we're after and if we don't find it we put that page back pull in the next one put that one back pull in the next one until we eventually find it and so as you're starting to realize this is a slow process to illustrate this if we had a 100 million users in our user table and each page as I said has maybe about 100 rows then that's a million of these Pages now each round trip from SSD to Ram is about 100 micros seconds and so that's 100 seconds in the worst case for us to find the item that we're looking for now it's worth noting that in reality there's pre-etching and there's other database optimizations that would probably actually get this down closer to 3 to 5 seconds but the point stands this is far longer than a user wants to wait in order to get get a query for some data for a given input ID so how do indexes solve this problem for us well indexes are just data structures that are stored on disk and act as a map to tell us where items or on which page items exist in the database and so when a new query comes in for a particular item we first pull that index into memory check which page that index tells us the resource or the item lives on and then we pull only that particular page in this case maybe it was page one and so as opposed to reading up to a million Pages as was the cas case with our sequential or table scan now we use the index to just tell us exactly which page we should look at so now naturally the question becomes well what types of indexes are there and when should I use them let's start by first explaining what is by far the most popular database index which is the B tree and so B trees are just basic tree structures just like what you've learned in data structures and algorithms where each node in the tree is a sorted list of values with pointers to another page in disk either a child node as the case here or an actual data page like here and so let's illustrate this with an example I think it'll make much more sense so let's say that we had a query like this where we want to select all users where their age is exactly 51 and in our user table to do this we built an index on age and so this is the index on age these are all the different ages of our users in our users table and so the first thing that we would do is we would pull in our root node into memory so if we go back up to our example up here that root node itself each one of these blue boxes is a page on disk and so we'd bring that guy into memory we would look at it and we'd say we want 51 so that's greater than 50 less than 90 so we're going to pull in this page next that would pull in a different index page and then we'd go in here and look at it and say well 51 is less than 55 and so which page does that correspond to it corresponds to page three and so we'd come up here and we' pull page three into memory and that's where all of the users that are age 51 exist now what if we were to do something like this and we would change this from an uh a Direct Value lookup to a range query well in this case we're going to pull in our index look at our root and we're going to pull in both of these blocks into memory this time this one and this one because they're both greater than 50 and now we're going to follow the pointers in each of these so all seven of them and we're going to load all seven of those pages into memory because that's all of our users who are greater than 51 

The next index worth discussing is called a hash index 
and so hash index is really straightforward it's just a hashmap 

so again that exists in disk 

and if you're looking for for example a user with a given email you're going to pass that email into a hash function and then you're going to have some hashmap that maps that key that hashed email to a value where the value is just a pointer of where that data exists on disk 

and so for example in this case the data for John his full row exists on page four and so we're going to pull that into memory 

Now in reality and this is important hash indexes are rarely actually used in production databases so while they offer 01 lookups which is great um B trees perform nearly just as well for exact matches but they also support those range queries and sorting that we were looking at just a moment ago 

and so the only Places You'll commonly see these hash indexes are for inmemory stores. Things like Redis where dis IO patterns aren't really that relevant 

so it's good to know from a historical con context perspective it's useful for caching and other things but typically in an actual database you're going to offer a b tree not a hash index okay so we get it B trees Rock and they are great for the majority of use cases 

but where instances especially in a system design interview where you wouldn't want to use a b tree 

well the first place is anytime that you have geospatial data specifically if you're trying to search within regions on latitude and longitude 

So think of a system design interview like design yel or find my nearby friends or something you want everybody within a given radius of something and so let's talk about why B trees don't work really well in this case so if I had a query that was something like this I want all of the locations that have a Latitude greater than 100 less than 400 and a longitude greater than 20 less than 200 so be trees really excel at onedimensional data but not two-dimensional data like this and the reason is really easy to illustrate and so if we look at this diagram here that query would give us all of this data for latitude all of this data for longitude and then we're going to load both of those things into memory and then do a fairly expensive merge until we get this middle area which is what we actually care about and so we still needed to get each of these long strips fold them into memory and do the merge is there a more efficient way to do that and the answer is yes there are specialized indexes or sometimes just algorithms before the indexing called geospatial indexing and the three most popular 

And the three best to know for system design interviews are one geohashing two quad trees and three R trees 

and we're going to really briefly go over each of these. So first up. let's look at geohashing. 
with geohashing you take the map of the World Image that's what this is and we split it into four parts we label the top left zero then one then two then three.

Now we can then recursively split each of those cells and do the same thing, such that this one is now 20 21 22 23 
and by continuing to do this we get increasing levels of precision 

such that maybe the New Mexico area is 31 but Albuquerque specifically is 310 

Now the nice thing is once you've converted latitude and longitude into these one-dimensional strings here um then now all nearby locations share a similar prefix and so this is Chihuahua I guess if we want to find everything near Chihuahua at this degree of precision then we just get the geohashes near it 321 331 312 and 332 and they all share similar prefixes and so what we do is we actually create the geohashes and then just build a b Tree on top of of the hashes themselves which allow us to easily do these range queries and these o of one or nearly o of one lookups on precise areas um and that's how geohashing works now it's worth noting that I did this as just simple numbers for the illustration we actually base 32 and code them and so for example Los Angeles where I'm at is this that's the geohash for Los Angeles 


Next up is quad trees it's pretty similar to geohashing and that we split the world recursively but there are a couple subtle differences the first is that we map this recursive splitting actually to a tree not a onedimensional string and we only need to go deeper in this tree in the places where we have high density so let me show you what this means imagine this was a map of the world and each dot is a location in our database say this was Yelp each dot is a business and so we split the world into four grids we thus create a tree where the tree has four children and that points to all of the businesses within that grid so all of the red businesses all the green businesses is this node blue and yellow respectively now you'll notice there's a lot of density in the blue cell here and so with quad trees you specify a k value which is basically saying that if any cell has greater than K value items then recursively split and so that was the case here we had more than five so we split it again and we can come over here and we had another four children right one two 3 four and then one such children or child still had more than five so we split it again and so 1 2 3 four right and so now when we want to find a particular business we just work our way down the tree accordingly and this is the index similar to the B tree that ends up being stored on disk 


lastly we have R-Trees. R-Trees are derived from quad trees. 
They're really similar concept but instead of just crudely splitting the world into even fours, it's a bit more Dynamic. 

it does some clustering in order to find locations or businesses or whatever it may be in your database that are close to each other 
and then each of these kind of larger groupings can even have a little bit of of overlap 
and so it's the same general idea we work down a tree in order to get increasing Precision from M 

this larger box to I this box and then finally to all the businesses or locations in F uh 

it's just not the case that we need to be so stringent in splitting exactly in force 
it's a bit more Dynamic and obviously fairly complex and not something we're going to go into detail in this video what's important to note as we zoom out and look at all three of these strategies is that geohashing is very popular today it exists in things like reddis it's the default um many production databases rely on geohashing it's a fantastic option it's quick it gets to rely on database indexes that already exist like B trees and you just have an algorithm on the front end 

quad trees were foundational into the development of geospatial indexes but are actually not really used in production much nowadays 
instead their predecessor R-trees are what used in production 
and so like postgis the extension that enables geospatial indexing on postgress for example uses archeries 

and so the important thing is that in a system design interview if you get tasked with a situation where you have two dimensional data specifically on latitude and longitude then you know that you're going to want to introduce some geospatial index and you might mention to your interviewer that you understand the difference between each of these uh and specify the one that you want to go with in that particular situation 

let's look at another place where B-Trees might not be a good choice for your database index 

in this case we want to select to all businesses that have pizza in the name 

now the reason that a B-Tree is not great here is because if you remember B-trees were sorted and so in the case of strings this means they're sorted lexicographically and so this would be great if it was a prefix search 

if we were looking for things that started with pizza well your B-Tree is awesome 

but if we want things that have pizza anywhere in the name, well, we have no choice but to do that dreaded Full Table scan again 
we have to pull in every single page and look at all of them 

and so instead what we should do is we should create something called an inverted index and so inverted indexes are great anytime that you need to search for text 

and so this is how we would do it. you can imagine that we have three documents.  

document one which says berries are fast and reliable. 
document two hash tables are fast but limited. 
doc three B-Trees handle range querys well 
You get the picture 

now what we can do is we can create a map, a hash map mapping each of the words that appear or the tokens that appear to all of the documents that they appear in 

and so now now if this was maybe something like Fast uh we want to get all of the documents in this example that we have down here that have the word fast in it well this is easy we're just going to look up fast and then find all the documents that map to fast and this is going to be up maybe pointers to the pages that these obviously exist in memory or on disk and we'll pull those into memory and then uh return those rows 

so this is inverted indexes inverted indexes used in things like elastic search postgress full text search anytime you're doing full text search you want to inverted indexes in your system design interview if you need to search over something with full text you'll want to mention uh that you'll need one of those Technologies which supports an inverted index 


okay so to wrap up I have a confession to make it's not likely in your actual interview that you're going to get deep into the implementation of each of these indexes. 

instead what's most important that you understand where your queries might be inefficient 
which columns you should apply indexes to and 
then depending on the case if there is a special index that needs to be applied 

and so this is a useful flowchart that that you should keep in mind. 
the first is do you need efficient data access well if the answer is no Then Full Table scans sure that's fine if it's yes then do you have a lot of rows in your table if you don't then you can stick to the Full Table scan it's not a lot of work to pull those pages into memory and read from them but if it's a yes then you have a simple question and I'm going to zoom in here and that's the what type of data are you querying if you're querying for text Data then you're going to want an inverted index elastic search Lucine full teex search and postgress these all will give you an inverted index if you're searching for location data then you need a geospatial index so reddis with geohashing post extension on postgress elastic searches implementation which I think uses a combination of geohash and archery don't quote me on that um do you need exact matches in memory that are fast well maybe consider a hash index um but also be be wary that maybe a b tree is still the best option and then for literally everything else go with a B tree so that wraps up our really quick Deep dive into database indexing we'll do more of these short-term form videos we'll also go back to the long form as well but let me know in the comments what you think if you have any questions and I'll see you all soon good luck with your interviews