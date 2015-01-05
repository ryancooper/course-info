# 6.830 Lab 3: B+ Tree Index

**Assigned: Wednesday, October 1, 2014**<br>
**Due: Wednesday, October 15, 2014 11:59 PM EDT**

## 0. Introduction



In this lab you will implement a B+ tree index for efficient lookups and range scans. We supply you with all of the low-level code you will need to implement the tree structure. You will implement searching, splitting pages, redistributing tuples between pages, and merging pages.


    
You may find it helpful to review sections 10.3--10.7 in the textbook, which provide detailed information about the structure of B+ trees as well as pseudocode for searches, inserts and deletes.



As described by the textbook and discussed in class, the internal nodes in B+ trees contain multiple entries, each consisting of a key value and a left and a right child pointer. Adjacent keys share a child pointer, so internal nodes containing *m* keys have *m*+1 child pointers. Leaf nodes can either contain data entries or pointers to data entries in other database files. For simplicity, we will implement a B+tree in which the leaf pages actually contain the data entries. Adjacent leaf pages are linked together with right and left sibling pointers, so range scans only require one initial search through the root and internal nodes to find the first leaf page. Subsequent leaf pages are found by following right sibling pointers. Left sibling pointers are not used in this implementation, but they are included for consistency with the textbook.



<!-- You may remember that B+ trees can prevent phantom tuples from showing up between two consecutive range scans by using next-key locking. Since SimpleDb uses page-level, strict two-phase locking, protection against phantoms effectively comes for free if the B+ tree is implemented correctly. We test next-key locking in our test cases to confirm your implementation is correct. -->

This is the first time we are running this lab, so please be understanding if something isn't clear or you find bugs in the code. As of right now, B+ trees are not hooked up to to the Query Planner you implemented in Lab 2, but in the future we are planning to more tightly integrate B+ trees into SimpleDB.  Thanks for being our guinea pigs; feedback is appreciated!

## 1. Getting started 

You should begin with the code you submitted for Lab 2 (if you did not
submit code for Lab 2, or your solution didn't work properly, contact us to
discuss options).  Additionally, we are providing extra source and test files 
for this lab that are not in the original code distribution you received. 


You will need to add these new files to your release. The easiest way
to do this is to change to your project directory (probably called simple-db-hw) 
and pull from the master GitHub repository:

```
$ cd simple-db-hw
$ git pull upstream master
```


## 2. Search

Take a look at `BTreeFile.java`. This is the core file for the implementation of the B+Tree and where you will write all your code for this lab. Unlike the HeapFile, the BTreeFile consists of four different kinds of pages. As you would expect, there are two different kinds of pages for the nodes of the tree: internal pages and leaf pages. Internal pages are implemented in `BTreeInternalPage.java`, and leaf pages are implemented in `BTreeLeafPage.java`. Additionally, header pages keep track of which pages in the file are in use, and are implemented in `BTreeHeaderPage.java`. Lastly, there is one page at the beginning of every BTreeFile which points to the root page of the tree and the first header page. This singleton page is implemented in `BTreeRootPtrPage.java`. Familiarize yourself with the interface of these four classes, especially `BTreeInternalPage.java` and `BTreeLeafPage.java`. You will need to use these classes in your implementation of the B+Tree.


Your first job is to implement the `findLeafPage()` function in
`BTreeFile.java`. This function is used to find the appropriate leaf page given a particular key value, and is used for both searches and inserts. For example, suppose we have a B+Tree with two leaf pages. The root node is an internal page with one entry containing one key and two child pointers. Suppose the single entry has a key value of 6. Given a value of 1, this function should return the first leaf page. Likewise, given a value of 8, this function should return the second page. The less obvious case is if we are given a key value of 6.  There may be duplicate keys, so there could be 6's on both leaf pages. Think carefully about how this function should behave when we are given a key value equal to one of the entries in the internal nodes.
    

Your `findLeafPage()` function should recursively search through internal nodes until it reaches the leaf page corresponding to the provided key value. In order to find the appropriate child page at each step, you should iterate through the entries in the internal page and compare the entry value to the provided key value. `BTreeInternalPage.iterator()` provides access to the entries in the internal page using the interface defined in `BTreeEntry.java`. This iterator allows you to iterate through the key values in the internal page and access the left and right child page ids for each key. 

    
Your `findLeafPage()` code must also handle the case when the provided key value f is null.  If the provided value is null, return the left-most child every time in order to find the left-most leaf page. Finding the left-most leaf page is useful for scanning the entire file. Once the correct leaf page is found, you should return it.  You can check the type of page using the `pgcateg()` function in `BTreePageId.java`. You can assume that only leaf and internal pages will be passed to this function.
    

Remember to use the BufferPool `getPage()` function to get each internal page and leaf page. Every internal (non-leaf) page your implementation visits should be fetched with READ_ONLY permission, except the returned leaf page, which should be fetched with the permission provided as an argument to the function.  These permission levels will not matter for this lab, but they will be important for the code to function correctly in future labs.

***

**Exercise 1: BTreeFile.findLeafPage()**

  
  Implement BTreeFile.findLeafPage().

  
  After completing this exercise, you should be able to pass all the unit tests in `BTreeFileReadTest.java` and the system tests in `BTreeScanTest.java`.

***


## 3. Insert

In order to keep the tuples of the B+Tree in sorted order and maintain the integrity of the tree, we must insert tuples into the leaf page with the enclosing key range. As was mentioned above, `findLeafPage()` can be used to find the correct leaf page into which we should insert the tuple. However, each page has a limited number of slots and we need to be able to insert tuples even if the corresponding leaf page is full.
    

As described in the textbook, attempting to insert a tuple into a full leaf page should cause that page to split so that the tuples are evenly distributed between the two new pages. Each time a leaf page splits, a new entry corresponding to the first tuple in the second page will need to be added to the parent node. Occasionally, the internal node may also be full and unable to accept new entries. In that case, the parent should split and add a new entry to its parent. This may cause recursive splits and ultimately the creation of a new root node. 


In this exercise you will implement `splitLeafPage()` and `splitInternalPage()` in `BTreeFile.java`. If the page being split is the root page, you will need to create a new internal node to become the new root page, and update the BTreeRootPtrPage. Otherwise, you will need to fetch the parent page with READ_WRITE permissions, recursively split it if necessary, and add a new entry.  You will find the function getParentWithEmptySlots() extremely useful for handling these different cases.  In `splitLeafPage()` you should "copy" the key up to the parent page, while in `splitInternalPage()` you should "push" the key up to the parent page. Review section 10.5 in the text book if this is confusing. Remember to update the parent pointers of the new pages as needed. When an internal node is split, you will need to update the parent pointers of all the children that were moved. You may find the function `updateParentPointers()` useful for this task. Additionally, remember to update the sibling pointers of any leaf pages that were split. Finally, return the page into which the new tuple or entry should be inserted, as indicated by the provided key field. 
    

Whenever you create a new page, either because of splitting a page or creating a new root page, call `getEmptyPage()` to get the page number of the new page. This function is an abstraction which will allow us to reuse pages that have been deleted due to merging (covered in the next section).


In both `splitLeafPage()` and `splitInternalPage()`, you will need to update the set of dirtyPages with any newly created pages as well as any pages modified due to new pointers or new data. This could be a large set of pages, especially if any internal pages are split. As you may remember from previous labs, the set of dirty pages is returned to prevent the buffer pool from evicting dirty pages before they have been flushed.

***

**Exercise 2: Splitting Pages**


  
  Implement BTreeFile.splitLeafPage() and BTreeFile.splitInternalPage().

  After completing this exercise, you should be able to pass the unit tests in `BTreeInsertTest.java`. You should also be able to pass the system tests in `systemtest/BTreeInsertTest.java`.  Some of the system test cases may take a few seconds to complete. These files will test that your code inserts tuples and splits pages correcty, and also handles duplicate tuples.

<!-- After completing this exercise, you should be able to pass the unit tests in `BTreeDeadlockTest.java` and `BTreeInsertTest.java`. Some of the test cases may take a few seconds to complete. `BTreeDeadlockTest` will test that you have implemented locking correctly and can handle deadlocks. `BTreeInsertTest` will test that your code inserts tuples and splits pages correcty, and also handles duplicate tuples and next key locking. -->

***

  


## 4. Delete

In order to keep the tree balanced and not waste unnecessary space, deletions in a B+Tree may cause pages to merge. You may find it useful to review section 10.6 in the textbook.
    


As described in the textbook, attempting to delete a tuple from a leaf page that is less than half full should cause that page to either steal tuples from one of its siblings or merge with one of its siblings.  If one of the page's siblings has tuples to spare, the tuples should be evenly distributed between the two pages, and the parent's entry should be updated accordingly. However, if the sibling is also at minimum occupancy, then the two pages should merge and the entry deleted from the parent. Occasionally, deleting an entry from the parent will cause the parent to become less than half full. In this case, the parent should steal entries from its siblings or merge with a sibling. This may cause recursive merges or deletion of the root node if the last entry is deleted from the root node. 


In this exercise you will implement `stealFromLeafPage()`, `stealFromLeftInternalPage()`, `stealFromRightInternalPage()`, `mergeLeafPages()` and `mergeInternalPages()` in `BTreeFile.java`. In the first three functions you will implement code to evenly redistribute tuples/entries if the siblings have tuples/entries to spare. Remember to update the corresponding key field in the parent. In `stealFromLeftInternalPage()`/`stealFromRightInternalPage()`, you will also need to update the parent pointers of the children that were moved. You should be able to reuse the function `updateParentPointers()` for this purpose. 
    

In `mergeLeafPages()` and `mergeInternalPages()` you will implement code to merge pages, effectively performing the inverse of `splitLeafPage()` and `splitInternalPage()`.  You will find the function `deleteParentEntry()` extremely useful for handling all the different recursive cases.  Be sure to call `setEmptyPage()` on deleted pages to make them available for reuse.

***

**Exercise 3: Redistributing and merging pages**


  
  Implement BTreeFile.stealFromLeafPage(), BTreeFile.stealFromLeftInternalPage(), BTreeFile.stealFromRightInternalPage(), BTreeFile.mergeLeafPages() and BTreeFile.mergeInternalPages().

  
  After completing this exercise, you should be able to pass the unit tests in `BTreeFileDeleteTest.java`.  You should also be able to pass the system tests in `systemtest/BTreeFileDeleteTest.java`.  The system tests may take several seconds to complete since they create a large B+ tree in order to fully test the system. 
<!--and the system test in `BTreeTest.java`. Please note that the test in `BTreeTest.java` may take up to a minute to complete.-->

***


## 5. Logistics 

You must submit your code (see below) as well as a short (1 page, maximum)
writeup describing your approach.  This writeup should:



*  Describe any design decisions you made, including anything that was
difficult or unexpected.

*  Discuss and justify any changes you made outside of BTreeFile.java.

*  Optional: How long did this lab take you? Do you have any suggestions for ways to improve it?



### 5.1. Collaboration 

This lab should be manageable for a single person, but if you prefer
to work with a partner, this is also OK.  Larger groups are not allowed.
Please indicate clearly who you worked with, if anyone, on your writeup.  

### 5.2. Submitting your assignment 


<!-- To submit your code, please create a <tt>6.830-lab7.tar.gz</tt> tarball (such
that, untarred, it creates a <tt>6.830-lab7/src/simpledb</tt> directory with
your code) and submit it for the Lab 7 assigment on the  <a
href="https://stellar.mit.edu/S/course/6/sp13/6.830/homework/">Stellar Site Homework Section</a>.
You may submit your code multiple times; we will use the latest version you
submit that arrives before the deadline (before 11:59pm on the due date).  If
applicable, please indicate your partner in your writeup.  Please also submit
your individual writeup as a PDF or plain text file (.txt).  
Please do not submit a .doc or .docx. -->

You may submit your code multiple times; we will use the latest version you
submit that arrives before the deadline (before 11:59 PM on the due date). 
Place the write-up in a file called <tt>answers.txt</tt> or 
<tt>answers.pdf</tt> in the top level of your <tt>simple-db-hw</tt> directory.  **Important:** 
In order for your write-up to be added to the git repo, you need to explicitly add it:

```bash
$ git add answers.txt
```
You also need to explicitly add any other files you create, such as new *.java 
files.

The criteria for your lab being submitted on time is that your code must be
**tagged** and 
**pushed** by the date and time. This means that if one of the TAs or the
instructor were to open up GitHub, they would be able to see your solutions on
the GitHub web page.

Just because your code has been commited on your local machine does not
mean that it has been **submitted**; it needs to be on GitHub.

There is a bash script `turnInLab3.sh` in the root level directory of simple-db-hw that commits 
your changes, deletes any prior tag
for the current lab, tags the current commit, and pushes the branch and tag 
to GitHub.  If you are using Linux or Mac OSX, you should be able to run the following:

   ```bash
   $ ./turnInLab3.sh
   ```
You should see something like the following output:

 ```bash
 $ ./turnInLab3.sh 
[master b155ba0] Lab 3
 1 file changed, 1 insertion(+)
Deleted tag 'lab3' (was b26abd0)
To git@github.com:MIT-DB-Class/hw-answers-becca.git
 - [deleted]         lab3
Counting objects: 11, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (6/6), 448 bytes | 0 bytes/s, done.
Total 6 (delta 3), reused 0 (delta 0)
To git@github.com:MIT-DB-Class/hw-answers-becca.git
   ae31bce..b155ba0  master -> master
Counting objects: 1, done.
Writing objects: 100% (1/1), 152 bytes | 0 bytes/s, done.
Total 1 (delta 0), reused 0 (delta 0)
To git@github.com:MIT-DB-Class/hw-answers-becca.git
 * [new tag]         lab3 -> lab3
```


If the above command worked for you, you can skip to item 6 below.  If not, submit your solutions for lab 3 as follows:

1. Look at your current repository status.

   ```bash
   $ git status
   ```

2. Add and commit your code changes (if they aren't already added and commited).

   ```bash
    $ git commit -a -m 'Lab 3'
   ```

3. Delete any prior local and remote tag (*this will return an error if you have not tagged previously; this allows you to submit multiple times*)

   ```bash
   $ git tag -d lab3
   $ git push origin :refs/tags/lab3
   ```

4. Tag your last commit as the lab to be graded
   ```bash
   $ git tag -a lab3 -m 'lab3'
   ```

5. This is the most important part: **push** your solutions to GitHub.

   ```bash
   $ git push origin master
   $ git push origin lab3 
   ```

6. The last thing that we strongly recommend you do is to go to the
   [MIT-DB-Class] organization page on GitHub to
   make sure that we can see your solutions.

   Just navigate to your repository and check that your latest commits are on
   GitHub. You should also be able to check 
   `https://github.com/MIT-DB-Class/hw-answers-(your student name)/tree/lab3`


#### <a name="word-of-caution"></a> Word of Caution

Git is a distributed version control system. This means everything operates
offline until you run `git pull` or `git push`. This is a great feature.

The bad thing is that you may forget to `git push` your changes. This is why we
strongly, **strongly** suggest that you check GitHub to be sure that what you
want us to see matches up with what you expect.


<a name="bugs"></a>
### 5.3. Submitting a bug 

SimpleDB is a relatively complex piece of code. It is very possible you are going to find bugs, inconsistencies, and bad, outdated, or incorrect documentation, etc.

We ask you, therefore, to do this lab with an adventurous mindset.  Don't get mad if something is not clear, or even wrong; rather, try to figure it out
yourself or send us a friendly email. 
Please submit (friendly!) bug reports to <a
href="mailto:6.830-staff@mit.edu">6.830-staff@mit.edu</a>.
When you do, please try to include:



* A description of the bug.

* A <tt>.java</tt> file we can drop in the `src/simpledb/test`
directory, compile, and run.

* A <tt>.txt</tt> file with the data that reproduces the bug.  We should be
able to convert it to a <tt>.dat</tt> file using `PageEncoder`.



You can also post on the class page on Piazza if you feel you have run into a bug.

<a name="grading"></a>
### 5.4 Grading 

50% of your grade will be based on whether or not your code passes the
system test suite we will run over it. These tests will be a superset
of the tests we have provided; the tests we provide are to help guide
your implementation, but they do not define correctness.
Before handing in your code, you should
make sure it produces no errors (passes all of the tests) from both
<tt>ant test</tt> and <tt>ant systemtest</tt>.



**Important:** Before testing, we will replace your <tt>build.xml</tt>,
<tt>HeapFileEncoder.java</tt>, <tt>BTreeFileEncoder.java</tt>, and the entire contents of the
<tt>test/</tt> directory with our version of these files.  This
means you cannot change the format of the <tt>.dat</tt> files!  You should
also be careful when changing APIs and make sure that any changes you make 
are backwards compatible. In other words, we will
pull your repo, replace the files mentioned above, compile it, and then
grade it. It will look roughly like this:

```
$ git pull
[replace build.xml, HeapFileEncoder.java, BTreeFileEncoder.java and test]
$ ant test
$ ant systemtest
[additional tests]
```

If any of these commands fail, we'll be unhappy, and, therefore, so will your grade.



An additional 50% of your grade will be based on the quality of your
writeup and our subjective evaluation of your code.



We've had a lot of fun designing this assignment, and we hope you enjoy
hacking on it!


