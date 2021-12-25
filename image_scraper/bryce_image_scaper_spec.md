# Design An Image Index for searching through images


## What do i mean by image index?
-  I want you to create a system to store all easily accessible new images posted on the web each day and build an application that lets a user search through all of those images.


### FUN FACT: WE CREATED 2.5 QUINTILLION data bytes every day in 2020







# Major components that need to be designed

1. Image Classifer
2. Web Crawler
3. Image Persistence and Storage




Lets start with the image persistence and storage as well as some back of the envelope calculations.



FIrst lets start with some estimations
1. We store 5 billion new photos every day
2. Each File has an average size of 1 mb
We now know we want to store 500,000 GB to store 5 billion photos. Lets make some more assumptions on how our service will be used.
3. We have 1 billion daily active users.
4. Each user will search for 10 terms a day which will amount to 1,000 photos each day
We know that we will be querying 1e+12 photos each day. We also know that since the average user will search 10 terms and an average of 1,000 photos we can group photos into similar groups based on what is searched.
We can store trends data on searches and move those files onto ssd since they are being queried at the highest rate. so maybe that image reading layer can re-index items based on popularity? Or perhaps the re-indexing of all of our images becomes too expensive.

#### Indexing our images
So that raises the question, how do we want to go about indexing our images? What do we want to store for each image?

1. imageLocation aka what machine what disk sector, a way to look up those images.
2. Image Tags, with relevance ```['goose',.98]``` This way we are 98 percent certain this image is a goose 

If we tag all of our images we can simply look them up by tag, since our search queries want a group of images that are similar to the tag. This will require some sort of fuzzy search algorithm.
If we want to define an algorithm to convert our search to a similar term, we can maybe use some sort of string metric algortihm that rates how similar this string is to other tags in the db. Also if our search index doesnt have many similar search terms we want to add it as a new term and run a classification on our images

To learn more about fuzzy search read [link](https://stackoverflow.com/questions/32337135/fuzzy-search-algorithm-approximate-string-matching-algorithm)

If we handle the image search this way it becomes much more simplistic than running a nlp classification on all of our images and searching which images in our index are most similar to our search term. We could do this but I am not all of google and am not capable enough. 

Now lets talk about how we can easily reference certain tags. I was thinking the flow would look like
```
Rough Term -> fuzzySearch(roughTerm) -> existingTermInSearchIndex -> Return the top 100 most relevant images that match searchTerm
```

If the flow looks like this we are still missing 3 steps in this search process
1. Getting the documents with a given term in the search index
2. Filtering the top 100 relevant images

Maybe we can use some sort of Network model that has links to other nodes in the network and a maximum heap of relevance for each search term, this will trade space for time, but with many more reads than writes I believe this is a acceptable tradeoff.


We can use a hashindex to keep reference of where each search term heap lives

```
{
TERM_1: [maxheap],
TERM_2: [max heap sorted by the relevance to this term. The capicacity of the heap will be 1000 until a user asks for more than a thousand images.]
}
```
So we have an immediate reference to each search terms images that are relevant. The bottle neck is the storage. We need to define another estimation to do a back of the envelope calculation for the space this will take.

#### How many search terms exist?
Lets just estimate that we have a start of 333,333 terms as this is how many english words are in the csv dataset i used for a random search index project.
Everyday lets say we add 1000 new search terms.

So this search engines size will grow everyday in size. Lets say the average search term is 8 unicode characters long, and the average unicode character takes 1 to 4 bytes [according to](https://stackoverflow.com/questions/5290182/how-many-bytes-does-one-unicode-character-take).

Our growth equation will be something like 
```
def size_per_day(num_of_days):
	return num_of_days * 1000
initial_search_terms = 333333
search_index_size = initial_search_terms + size_per_day(num_of_days_passed)
```
The initial terms will take up 333333 * 8 * 2 = 5,333,328 bytes if the average byte size of a unicode character is 2.
The additional state will grow 1000 terms which average in 8 characters and 2 bytes a character = 16,000 bytes a day.

now we need to calculate the size the maxheaps will take. To do that we need to define how much space the average item in a max heap will take.which requires definition of our heap items

```python
class HeapItem:
    def __init__(self, imagePointer, term_pair):
	self.imagePointer = imagePointer
	self.term_pair = term_pair # ['term', 'relevance_percentage'] 

```
 
That handles the ideas of searching but what about storage? what exactly is our image pointer? 
