authorId: dlandis
title: Resizing a Numeric Column in a PostgreSQL Table
date: 2016-09-14 12:02:59
tags: PostgreSQL, Postgres, SQL, Databases
---

While using PostgreSQL, you may find yourself in a situation where you have a column whose data type is now too small and the length needs to be increased.  Updating a column type in PostgreSQL can, at times, be nothing short of very painful.

Let's say you have a column of type `varchar(25)`.  When you first created the column, you decided there was absolutely no way you could ever need more than 25 characters in that column.  Fast forward months or years later and you now realize that column requires 40 characters.  This would be fine, except that (A) the table is huge - which means it could take a significant amount of time for this command to finish - and (B) there are views and rules that depend on that column - which generates errors when you try the standard `ALTER TABLE my_table ALTER COLUMN my_column TYPE varchar(40);`.  There is no good solution to (A) other than waiting.  Which may or may not be allowed based on your business needs.  The solution to (B) is painfully manual.  You need to drop all dependent views and rules (e.g. Primary Key, etc), make the column data type change, then recreate all dependent views and rules.  This sucks.

Luckily, the folks over at [sniptools.com](http://sniptools.com) solved this exact problem in this [post](http://sniptools.com/databases/resize-a-column-in-a-postgresql-table-without-changing-data).  I won't go into the details (you should look at the post directly), but suffice it to say, I have used their solution multiple times on a production database and it has worked amazingly well.

Great.  Problem solved.  ...Except that now we have the exact same problem with columns of type `numeric(precision, scale)`.  I have a column of type `numeric(2,0)` and I really need it to be `numeric(4,0)`.  I'm running into all of the same problems as the `varchar` issue above.

Thankfully, there is a very similar solution!  To demonstrate this, let's start by creating a fake table of varying numeric types:
```
CREATE TABLE my_numeric_test(
   numeric_one_zero numeric(1,0),
   numeric_one_one numeric(1,1),
   numeric_two_zero numeric(2,0),
   numeric_two_one numeric(2,1),
   numeric_three_zero numeric(3,0),
   numeric_three_one numeric(3,1),
   numeric_four_zero numeric(4,0),
   numeric_four_one numeric(4,1),
   numeric_fortyfive_fifteen numeric(45,15),
   numeric_sixhundredeightythree_threehundred numeric(683,300)
);
```

Next, inspect the `atttypmod` of the different columns:
```
SELECT atttypmod, attname
FROM pg_attribute
WHERE 1 = 1
AND attrelid = 'my_numeric_test'::regclass
AND attname LIKE ('numeric_%')
ORDER BY atttypmod;
```

Notice, there is a pattern here:
`atttypmod` = `precision` * 65,536 + `scale` + 4

Let's say we want to update column `numeric_four_zero` to have type `numeric(9,0)`.  A couple of tests:
```
INSERT INTO my_numeric_test (numeric_four_zero) VALUES (1234);  -- works!
INSERT INTO my_numeric_test (numeric_four_zero) VALUES (123456789);  -- ERROR:  numeric field overflow
```

Using the algorithm from above, for `numeric(9,0)` we see `atttypmod` = 9 * 65,536 + 0 + 4 = 589,828.  Here is how we can update the column type:
```
-- Regular way to update column type (don't use this for this example...)
--ALTER TABLE my_numeric_test ALTER COLUMN numeric_four_zero TYPE numeric(9,0);

-- Hack way to update column type
UPDATE pg_attribute
SET atttypmod = 589828
WHERE attrelid = 'my_numeric_test'::regclass
AND attname = 'numeric_four_zero';
```

We can run the same test as above and see that it works:
```
INSERT INTO my_numeric_test (numeric_four_zero) VALUES (123456789);  -- works!
```

We can also select the column from the table and see that the column type has changed:
```
SELECT * FROM my_numeric_test;
```

Finally, cleanup:
```
DROP TABLE my_numeric_test;
```

Warning: I'm not sure if there are any side effects of doing this on your own code.  I _think_ it should work, but give no guarantees implicitly nor explicitly that it will not turn your database into a smoking, ruined heap.

Again, many thanks to this sniptools [post](http://sniptools.com/databases/resize-a-column-in-a-postgresql-table-without-changing-data).  Without them, it would not have been possible.

Hope that helps!

#### Update (Sept 19, 2016):
I was too worried about potential side effects of using this hack and opted to _not_ use it on a production environment.  Instead, I dropped 80 views, updated about 65 column data types, and then recreated the 80 views.  It required lots more work, but this way, I'm more confident in the final product.  As stated before, if you do use this hack, do so at your own risk.
