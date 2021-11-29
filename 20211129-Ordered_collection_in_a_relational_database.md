Ordered collection in a relational database
A short story about how this problem can be modeled in a relational database and when you can encounter it.
Photo by Markus Spiske on UnsplashThis article is targeted at engineers who want to learn something new about data modeling and writing SQL. It doesn't matter if you are a beginner or an advanced database user, I hope in both scenarios you will learn something new from this article. As I mentioned, it will be about modeling the real-world scenario where I will present to you some advanced queries built using SQL. Some of the elements can be Postgres specific, so please keep that in mind.
Drag and drop
You will probably need to order your collection when you have a drag and drop functionality in your app. One of the most common use cases is a kanban board. You have some space where you can shuffle your cards in any order you like.  The order is then persisted. If you put a card at the top of the todo column, anyone who visits the board should see it at the top of that column.
In one of our projects we were asked to prepare the functionality where journalists would be able to change the position of the content prepared by them. I won't be diving into too many details but the idea was to add drag and drop in the CMS (Content Management System) developed by us.  The work on the feature started in Figma where together with a UX designer we prepared sketches. We are using React with Material-UI as a foundation of our frontend components. For the drag and drop feature, we didn't want to reinvent the wheel and so we picked one of the many React libraries out there for that. We started thinking about the implementation on the backend and how we would  store the data in our Postgres database. The problem was not as trivial as we anticipated at the beginning.
Let's try to formulate our business requirements for this task using user stories. I will generalize the feature a little bit. From now on I will be using the kanban analogy (to be more specific: one column on a board) as it will be much easier to grasp than using the terminology we are using in our project. The core principles are the same.
As a user, I can add a new card
The new card will be placed at the end of the collection.
As a user, I can rearrange cards
Users can click and hold an item and drag it to any place in the collection. When moving it up all items between the current and the new position should be moved one place down (as demonstrated in the animation below). And vice-versa, when moving items down the new preceding items should go one spot up. When a position is changed only by one the two affected items should switch positions. The new arrangement of the items should be persistent.
As a user, I can delete a card
When deleted an item disappears from the collection. The position of the items that were below the deleted one is updated by one (up).
Database model - potential solutions
This isn't the first time when my team was handling a little bit more advanced data modeling.
You can find many complex SQL queries in our codebase used to fetch collections from the storage. In the Comments widget we provide ordering by best or most relevant items. The algorithms behind the mentioned sorting types use data that has already been stored before we started the implementation. Some examples are: the creation date, the number of replies, the number of up- or downvotes.
Surely, you are familiar with the ORDER BY clause. What value would you provide to the ORDER BY if you wanted to fetch cards in a persisted order 💡? Let's take a look at how we could model this problem in a relational database. Together with a team we were discussing three potential solutions.
Identifiers in the parent  -  array
```
# lists table
| id | cards              |
|----|--------------------|
| A  | [red, blue, green] |
# cards table
| id    | description  |
|-------|--------------|
|   red |         blah |
|  blue | As I user... |
| green |      foo bar |
```
We have two tables: a table lists and a table cards. One list can contain many cards, the relationship here is one-to-many. In the animations seen before we had one list, called List A with cards red, green, yellow, pink and blue. In the first of the considered solutions we modeled this as follows. We introduced the attribute cards in the table lists of the array type. This is one of the supported data types in Postgres, click here for more reading. This supports ordering out of the box. The card with the identifier red would be displayed at the top and green at the bottom. User's actions like adding or removing a card would require doing updates in both tables. Adding a new card would require inserting new records in the cards table and appending a new element at the end of the cards array in the lists table. To append an element to an array we could use array_append(list.cards, 'yellow') function and to remove array_remove(list.cards, 'red'). More about available operations on arrays can be found in the Postgres documentation. To rearrange cards you would first need to know what elements are in an array currently. On the client-side you would need to fetch the array, manipulate it and then pass the whole collection to a query where you would do something like SET cars = '{green, red, blue}'.
What is wrong with it?
We are using a relational database and if we want to use it optimally we should normalize our data. An association is considered normalized when a foreign key attribute is defined in the child table. The key points to the parent. With that in place we are able to do joins. Without that getting all cards for a given list will require making some nasty things in the SELECT clause. Also, as stated before, updates require manipulating data in both tables, and also the client needs to know the contents of the collection prior to that. Passing around the whole collection on the client-side sounds like looking for trouble regarding data consistency, especially when updates are frequent and simultaneous. Quite ugly 😒.
Auxiliary attribute - placement
```
# lists table
| id |
|----|
|  A |
# cards table
| id    | description  | list_id | placement |
|-------|--------------|---------|-----------|
|   red |         blah |       A |         1 |
|  blue | As I user... |       A |         2 |
| green |      foo bar |       A |         3 |
```
The values of the placement attribute represent the position on the list. 1 is the top whereas 3 represents the bottom. So looking from the top of our column we have: the red card, the blue card, and the green card. We also introduced the list_id attribute which is a link between the tables cards and lists. The data is normalized. We satisfied the first normal form. Now we don't need to manipulate the data in both tables. Adding a new card will be just a simple insert with one additional task to do. We need to calculate the value of a placement for it. The formula is number_of_cards + 1 or placement_of_the_last_card + 1. Removing and rearranging will require updates in all records that are affected. What does it mean? Let's use an example: When we remove a card from the top, the second card now becomes first, the third becomes the second etc. All cards below the removed one are moving one position up. With a rearrangement things get a little bit more complicated. Rearranging, let's say the fifth card to the second place will result in all cards between the second and fourth place moving one position down. Check the animation from the previous section where the pink card was promoted above the green card. Besides promoting a card we also can degrade a card. This will happen in the opposite operation where we drag a card to the bottom, which is similar to removing it. When it comes to pulling out our data, it should be a pleasure, we just do ORDER BY placement and we are good to go. Pagination should be delightful.
What is wrong with that?
When rearranging cards we need to update all affected records, not only the record of the card that we are dragging around. To get all the affected records we need to iterate through all the cards, the complexity of such an operation is equal to O(n). We have two different directions to support (promoting and degrading), which can lead to quite a complicated update query. With such a query the probability of introducing a bug 🐛 is high. You can protect yourself from that by implementing different constraints on the table but building advanced constraints can also lead to bugs. We also need to remember that database tests are quite costly, so you won't have them or they will be limited. To sum up, you need to be proficient in writing advanced queries using SQL if you want to follow this approach.
Trying to be smart - rank
```
# lists table
| id |
|----|
|  A |
# cards table
| id    | description  | list_id   | rank      |
|-------|--------------|-----------|-----------|
|   red |         blah |         A |       100 |
|  blue | As I user... |         A |       200 |
| green |      foo bar |         A |       300 |
```
In terms of querying for data this solution is pretty similar to the previous one. To sort a collection we would use ORDER BY rank. The lowest value means that a given element is at the beginning of the list. A user would see the red card at the top, then the blue one, and the green one at the bottom. The differences appear when the user starts shuffling the cards, i.e. on updates. Let's explain this solution by looking at what the data looks like after a user moves the green card one position up.
```
| id    | description  | column_id | rank |
|-------|--------------|-----------|------|
|   red |         blah |         A |  100 |
| green |      foo bar |         A |  150 |
|  blue | As I user... |         A |  200 |
```
The value of the rank attribute for the green card is 150. How did this magical value 🦄 appear there? To calculate it we used the following formula: green.rank = (red.rank + blue.rank) / 2. We put the card between the red and the blue card. Only one record needed to be updated, the rest of them are not affected, the computational complexity of it is O(1). However, you need to know the rank of the cards adjacent to the card being dragged. We have the same situation on deletion, we just delete a given record. For adding a new card we follow a similar principle as for the previous solution: To calculate rank we will do: number_of_cards * 100 or rank_of_the_last + 100. I think you get the idea, you don't need to use 100 as a multiplier, you can use 10000000, it doesn't really matter. The idea is to have something that allows you to calculate the relative value between two elements. The solution was inspired by this answer on StackOverflow. There is an extension of this idea that is used in JIRA. The Atlassian devs called it lexorank. Instead of integers they are using strings to sort a collection. If you are interested in more details, check the video on youtube.
What is wrong with that?
We would have a limited number of available rearrange actions. Let's imagine that a user is doing the action mentioned before in an infinite loop. We calculated the rank of the relocated card, it is 150. The next values in between are: 125, 113, 107, 104, 102, 101, and boom 💣, we don't have any integers between 100 and 101. We could use a much higher initial value, but we would still have a limited number of moves. JIRA's ranking solution may have a way to bypass this limitation. I'm saying may, as to understand this algorithm I would need to spend some more time on it. I know that software engineers love building and implementing complex algorithms but do they equally love maintaining them? Not sure about that. I can imagine the face of a developer who revisits such a piece of code or data in two years' time… They would go like 😲.
The winner?
In our case, we don't expect a high number of elements in the collections, five is probably the maximum value. Also, rearranging will be a relatively rare action. That being said, the computational complexity of the second algorithm where the placement attribute was introduced would be almost identical to the third one, the one with the rank attribute. We can easily assume that O(n) ≈ O(1). We rejected the first solution immediately as it isn't normalized. The level of the trickiness of the SQL queries of the second and the third is similar. We found it compelling that absolute values of the placement are easier to understand than the rank. There is no need for any additional explanation when looking at the records in the database. The placement 2 means that a card is the second one, whereas the rank of 150 doesn't say anything without context. We picked the placement option. It was easier to grasp the idea behind it. Performance aspects are not always the most important ones.
Implementation
Now it's time to dive into details. Let's recall the structure of the table that we already saw one more time. There is one more attribute deleted_at. It appears in the presented queries. When the value is set, a card won't be visible anymore. The value is set when a user deletes a card. Keeping the affected records in the database on delete is a technique called soft delete. We need to keep all data created by users. This introduces one difficulty that will cause us a small problem later on.
```
| id    | description  | list_id | placement | deleted_at          |
|-------|--------------|---------|-----------|---------------------|
|   red |         blah |       A |         1 |                null |
|  blue | As I user... |       A |         2 |                null |
| green |      foo bar |       A |         3 |                null |
|  pink | not relevant |       A |         3 | 2021-11-20T15:54:30 |
| black |      move me |       A |         4 |                null |
```
We added a constraint. A pair of list_id and placement needs to be unique only when deleted_at IS NULL. Only one visible card can be the first one. Sounds reasonable, right?
```
CREATE UNIQUE INDEX cards_listid_placement_deletedat_unique
ON cards (list_id, placement)
WHERE deleted_at IS NULL
```
I have previously mentioned three user actions that we need to cover. Adding, removing, and rearranging cards. Let's see how they can be accomplished using SQL.
Insert
```
INSERT INTO cards (id, list_id, placement)
SELECT
  :id,
  :list_id,
  (SELECT MAX(placement) + 1 FROM cards WHERE list_id = :list_id AND deleted_at IS NULL)

 -- Example values
 -- :list_id = 'A'
 -- :id = 'pink'
```
To add a card at the end we need to know where the end is. The SELECT from the line 5 will return a 1x1 table with that value. MAX is an aggregate function that returns the highest value in a column. Treat :id and :list_id as bind parameters, if you would like to execute this query you should put some values in there 📝.
Delete
```
WITH placement_of_removed AS (
  SELECT placement
  FROM cards
  WHERE id = :id AND deleted_at IS NULL
)
UPDATE cards
SET
  placement = (
    CASE WHEN placement > (SELECT * FROM placement_of_removed)
      THEN placement - 1
      ELSE placement
    END
  ),
  deleted_at = (
    CASE WHEN id = :id
      THEN NOW()
      ELSE NULL
    END
  )
WHERE list_id = :list_id AND deleted_at IS NULL

-- Example values
-- :list_id = 'A'
-- :id = 'pink'
```

Things get interesting. At the beginning of the query we have the keyword WITH. Such a statement is called a CTE (Common Table Expressions). We use CTEs when we want to create an auxiliary table with an alias that then can be referenced in other places. I usually use them if I want to have a cleaner query or if I don't want to repeat myself. What will this CTE return? Again a 1x1 table, and in the cell you will find the placement value of the card that we want to remove.
What happens afterwards? We need to iterate through all the records (all cards assigned to a given list that have not been removed), set deleted_at to thee current timestamp (lines 14 -18) for the removed card 🗑, and promote all the cards that were below the removed one (lines 9–12). Promotion 👆 means that the value of placement needs to be decremented.
Rearrange
```
WITH helper AS (
  SELECT
    placement AS old_placement,
    CASE
      WHEN placement = :new_placement THEN 0
      WHEN placement > :new_placement THEN 1
      ELSE -1
    END AS direction
  FROM cards
  WHERE id = :id
)

UPDATE cards

SET
  placement = (
    CASE
      WHEN id = :id
        THEN :new_placement
      WHEN placement
        BETWEEN
          LEAST(old_placement, :new_placement)
          AND
          GREATEST(old_placement, :new_placement)
        THEN placement + direction
      ELSE
        placement
    END
  )

FROM cards c JOIN helper h ON TRUE
WHERE list_id = :list_id AND deleted_at IS NULL

-- Example values
-- :list_id = 'A'
-- :new_placement = 3
-- :id = 'pink'
```

Whoah 😨! A lot is happening there. Let's try to decompose the query. What will the result be of the CTE aliased helper? Do you remember the table with five records from the beginning of the section? Let's imagine that a user would like to move the black card to the top of the list 🔝. The backend received the following request…
```
curl --request PATCH --data '{"placement":1}' $HOST/cards/black
```
In this scenario result of the CTE is:
```
:id = 'black'

| old_placement | direction |
|---------------|-----------|
|             4 |         1 |
```
What is the meaning of the direction attribute? When we move a card up all cards between the old and the new position need to be downgraded 👇, so the value of their placement needs to be incremented by 1️⃣. When we move the card down, the cards in between are moving one position up ☝️. So, direction = 1 means we are moving them higher up, direction = -1 lower down, and direction = 0 we don't move the card. This is a verbal definition of things happening in the lines 4–10.
Joining the helper with the results of the WHERE (lines 31–32) would be:
```
| id    | list_id | placement | old_placement | direction |
|-------|---------|-----------|---------------|-----------|
|   red |       A |         1 |             4 |         1 |
|  blue |       A |         2 |             4 |         1 |
| green |       A |         3 |             4 |         1 |
| black |       A |         4 |             4 |         1 |
```
Using TRUE in the JOIN clause results in appending each record in the table cards to each record in the CTE helper. As the number of elements in the table helper is 1 we append this record to each card's record.
Let's dig more into lines 16–28. We need to iterate through 4 cards. First, we need to compute new values of placement for the affected cards. If we are processing the black card we change the placement value to a new one. The first case is covered. The second case catches the cards with 1 ≤ placement < 4 , so red (placement = 1), blue (placement = 2) and green (placement = 3 ). To them we apply: placement + direction. Let's add up the values for the green card and we have 3 + 1 = 4 . For cards not affected by our move we don't change the position, it is the third case.
This is what the table should look like after the last update:
```
| id    | list_id | placement          |
|-------|---------|--------------------|
|   red |       A |             (+1) 2 |
|  blue |       A |             (+1) 3 |
| green |       A |             (+1) 4 |
| black |       A | (:new_placement) 1 |
```
Full of confidence in our query we launched it. The result was 🔥:
```
duplicate key value violates unique constraint "cards_listid_placement_deletedat_unique"
```
We didn't understand what could be wrong with our query 🤷‍♂. I have decided to DROP the constraint and run the query to see if we have duplication anywhere.
```
DROP INDEX cards_listid_placement_deletedat_unique
```

After executing the query without the constraint we received what we anticipated. All values of placement were unique 🤔. After a few minutes of thinking, we asked ourselves a question. We had 4 cards, so 4 records to update. But what order were they executed in underneath? Were the records updated sequentially? Or maybe simultaneously? After doing some reading 📚, we found out that you can't assume anything here. Internally, Postgres decides what is best for it. It updates the rows in an arbitrary order. It didn't solve our problem but it led us to an obvious conclusion.
We calculated the placement of the black card as 1️⃣. Before the execution 1️⃣ has already been assigned to the red one. I know that I mentioned something about not assuming anything but let's assume something 😂. If the update of the black card happens before the update of the red card we will have two cards with the placement of 1️⃣ at that point in time. Now the error makes some sense. We went back to Postgres documentation and found that:
By default, a constraint check is applied on each update in the table, not at the end of the whole transaction.
How can we hold off validation to the end of the whole transaction? Luckily, Postgres give us the possibility to configure when the constraint checks are performed. We can configure it to be deferred. The documentation describes how it can be achieved. We followed the guide and prepared a query to create a new deferrable constraint. We added DEFERRABLE INITIALLY DEFERRED to the definition.
```
ALTER TABLE cards
ADD CONSTRAINT cards_listid_placement_deletedat_unique
UNIQUE (list_id, placement) WHERE (deleted_at IS NULL)
DEFERRABLE INITIALLY DEFERRED
```
We received a syntax error 🤬. I did some googling and found a StackOverflow question. You can't define a partial unique constraint, you can do it for an index, but not for a constraint. The proposal was to use the EXCLUDE constraint. This was something new for me.
```
ALTER TABLE cards
ADD CONSTRAINT cards_listid_placement_deletedat_unique_exclude
EXCLUDE (list_id WITH =, placement WITH =) WHERE (deleted_at IS NULL)
DEFERRABLE INITIALLY DEFERRED
```

The EXCLUDE constraint does a comparison of two records in a table. So if the following statements are true:
```
red.deleted_at == null && black.deleted_at == null
red.list_id == black.list_id
red.placement == black.placement
```

The database will throw a constraint violation error.
With the new constraint, everything started to work as we expected 🎉.
Summary
I have promised you a short journey but in the end it wasn't that short 😂. I'm glad you made it to the end 🙇. I hope that you've learned something new about building queries using SQL. As you saw, those bigger ones are not so scary after you decompose them. From my experience, I can say that it is worth working on optimizing queries as they can give you huge benefits at a small cost. Often you abstract them by using an ORM or some query builder. These tools make our life easier but sometimes we use them blindly, without any reflection. I'm even convinced that there are some teams who encountered the problem presented in this article and switched to an implementation on the client-side. You can move some of your business logic to SQL, don't be afraid of it. The language is very powerful. Doing things closer to the database will guarantee you data consistency as you are much closer to the data source. I encourage you to write more raw queries!
