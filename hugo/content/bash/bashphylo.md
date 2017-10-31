---
title: "Prepping Phylogenetic Data With BASH"
date: 2017-10-31T08:22:40+02:00
draft: true
---


## Introduction

General usage of shell environments are restricted to executing simple commands to complete tasks. So

Some of the previous demonstrations have shown additional functionality provided by default in the
shell environment. For instance, you were shown that it was not necessary to copy-paste/retype full
pathnames. You can use a shell variable to reference said pathname.  

```bash
MYDATAFOLDER=/mnt/lustre/users/username/data/myinputdata


#Full path
samtools stats /mnt/lustre/users/username/data/myinputdata/something.bam

#Variable

samtools stats ${MYDATAFOLDER}/something.bam
```

This is a very trivial example, but still very useful to keep things tidy.  It's easy to get lost.
Besides aesthetics, the use of a _variable_ is exceptionally useful.  We can use them to store
output from another command. Let's say you want to keep a collection of all the fasta files in a
current directory. 


```bash
ls *.fasta # our command, just to check
1.fasta 2.fasta 3.fasta # etc

MYFASTAFILES=$(ls *.fasta)

echo ${MYFASTAFILES}

```

Let's break this down a bit.  First, how do we define a variable and do we access the value of a
variable?  We define a variable as ``VARIABLENAME=SOMEVALUE``.  Where ``VARIABLENAME`` can be any
abitrary name that (I think) has to start with an non-numeric normal character. The ``SOMEVALUE``
part can be a number, ``VARIABLENAME=5``, a string (a string is a sequence of characters),
``VARIABLENAME="Goody bash"``, or the output of a command. For the last part, we can _capture_
command output by wrapping the command in ``$()`` (Note, some guides will tell you that you can
also use backicks, `` ` ``, but this is deprecated in favour of ``$()``). So the output you see when
you type ``ls *.fasta`` is now _redirected_ to the variable when you encapsulate as``$(ls
*.fasta)``.



### Manipulating variables

Variables are by definition not static entities and can easily be changed to something else.
Indeed, by using the expression ``${MYDATAFOLDER}/something.bam``, you are creating a new (unnnamed)
variable that combines the value of ``MYDATAFOLDER`` and ``/something.bam``. Values of variables can
be manipulated in interesting ways.  A lot of time  when dealing with many data files, you'll sit
with a situation where manual grouping of data according to sample/group/etc becomes really
difficult.  

Take a look at the following filenames:


```
036-2012P_S1.out
036-2024P_S2.out
036-2036P_S3.out
039-2002-0_S61.out
079-2003-5_S62.out
079-2004P_S5.out
079-2012P_S6.out
079-2024P_S7.out
079-2036P_S8.out
079-2036_S3.out
093-2001_S2.out
093-2002P_S10.out
093-2002_S4.out
093-2004P_S11.out
093-2012-0_S63.out
093-2012P_S12.out
093-2024-0_S64.out
093-2024P_S13.out
093-2036P_S14.out
102-2002-0_S65.out
102-2004P_S15.out
102-2012-0_S66.out
102-2012P_S16.out
102-2012_S5.out
102-2024-0_S67.out
102-2024P_S17.out
102-2036-0_S68.out
102-2048P_S19.out

```

The format of each filename is:

```
  036  -  2012P _   S1     .out
(group)-(sample)_(sampleid).(ext)
```

I would like to organize these files by moving all the files belogning to the same group into a new
folder.  I would like this folder to be named according to the group name.  Furthermore, the
filenames should only contain the sample name (without sample id) and a different extension, ".txt".

```
036-2012P_S1.out --> 036/2012P.txt
```

How remove everything after the group number? We know that the group number is always followed by a
dash (``-``).  There are actually multiple ways of doing this.  The ``cut`` command allows you to
extract columns from text files, given a delimeter (separator).  So we could technically use the dash as the
delimiter and extact the first column:

```bash
echo "036-2012P_S1.out" | cut -f 1 -d\-
036

#Store it

GROUPNAME=$(echo "036-2012P_S1.out" | cut -f 1 -d\-)
echo ${GROUPNAME}
036
```

But it's quite cumbersome to do this, not to mention inefficient.  For every file, we need to use
the "cut" just to get the prefix of a file (which is the group in this case). Alternative, we could
use built in BASH string substitution commands.  To remove a suffix, an implicitly be left with a
prefix:

```bash
${VARIABLE%%[pattern]}

FILENAME="036-2012P_S1.out"
GROUPNAME=${FILENAME%%-*}

echo ${GROUPNAME}
036

```

The ``%`` operator tell BASH to remove everything after and including that which matches
``[pattern]``.  The pattern in this case is the first dash and _everything_ after that. If you were
to play around with the command a bit, you'd see the following:

```
echo ${FILENAME%1*} # Remove everything after the last occurrence of "1"
036-2012P_S

echo ${FILENAME%%1*} # Remove everything after the first occurrence of "1"
```

The command may seem a little confusing, but fret not.  The search for a pattern actually starts
from the end of the line.  A single ``%`` indicates that everything gets deleted from the end of the
string up to the _first_ match.  A double ``%%`` indicates that everything gets deleted up to the
final match starting from the end of the line. 


So now we know how to generate the group name.  Next, we need to extract the sample name.  We need
to remove the ``group prefix`` and everything from the ``sample id`` onward.  Then add the ``.txt``
extension.  


```bash

SUFFIX=${FILENAME#*-}
echo ${SUFFIX}
2012P_S1.out

SAMPLE=${SUFFIX%%_*}

echo ${SAMPLE}
2012P

```

So the ``#`` operator removes everything before and including a match.  The example above removed
everything up to and including the first "-".  However, we still need to remove the trailing bits,
so the following command assigns the ``SAMPLE`` variable as the ``SUFFIX`` variable minus the pesky
suffix, i.e. ``_S1.out``.


Finally, we can construct the desired filename:

```bash

FINALNAME=${GROUPNAME}/${SAMPLE}.txt

echo ${FINALNAME}
```


Great, but how do we put this all together? We still have many files to go through and it would be a
bit slow to do this procedure for every single filename manually.  We can use loops


### For loops

For loops are loops that go through items in some sort of list.  With each iteration, this item
grabbed from the list can be used in whatever way you see fit.  We'll loop through the files in our
directory.

```bash

for file in *.out
	do 
		FILENAME=$(basename ${file}) # removes the path - not really needed here
		GROUPNAME=${FILENAME%%-*}
		SUFFIX=${FILENAME#*-}
		SAMPLE=${SUFFIX%%_*}
		FINALNAME=${GROUPNAME}/${SAMPLE}.txt

		echo "mkdir -p out/${GROUPNAME}; cp ${file} out/${FINALNAME}"
	done
```

This for loop won't actually copy the files to the folder. It just echoes the command.  It is
imperative that you do this before doing a task such as mass copying/moving, because you _will_ make
a mistake. And indeed, this for loop does contain some funky names that include both the sample name
and some random suffix. We note that there is a rogue dash for some of the samples:


```
mkdir -p out/568; cp 568-2001-0_S89.out out/568/2001-0.txt
                                                   ^^^
```
	



So this has to do with the ``SAMPLENAME`` variable.  So it seems we need to look for an ``_`` and a
``-``.  Change the ``SAMPLE=${SUFFIX%%_*}`` line to ``SAMPLE=${SUFFIX%%[_-]*``. The block brackets
say that the pattern can match either ``-`` or ``_`` in the beginning.


```bash

for file in *.out
	do 
		FILENAME=$(basename ${file}) # removes the path - not really needed here
		GROUPNAME=${FILENAME%%-*}
		SUFFIX=${FILENAME#*-}
		SAMPLE=${SUFFIX%%[_-]*}
		FINALNAME=${GROUPNAME}/${SAMPLE}.txt

		echo "mkdir -p out/${GROUPNAME}; cp ${file} out/${FINALNAME}"
	done
```

And this produces a much better output


```

mkdir -p out/036; cp 036-2012P_S1.out out/036/2012P.txt
mkdir -p out/036; cp 036-2024P_S2.out out/036/2024P.txt
mkdir -p out/036; cp 036-2036P_S3.out out/036/2036P.txt
mkdir -p out/039; cp 039-2002-0_S61.out out/039/2002.txt
mkdir -p out/079; cp 079-2003-5_S62.out out/079/2003.txt
mkdir -p out/079; cp 079-2004P_S5.out out/079/2004P.txt
mkdir -p out/079; cp 079-2012P_S6.out out/079/2012P.txt
mkdir -p out/079; cp 079-2024P_S7.out out/079/2024P.txt
mkdir -p out/079; cp 079-2036P_S8.out out/079/2036P.txt
mkdir -p out/079; cp 079-2036_S3.out out/079/2036.txt
mkdir -p out/093; cp 093-2001_S2.out out/093/2001.txt
mkdir -p out/093; cp 093-2002P_S10.out out/093/2002P.txt
mkdir -p out/093; cp 093-2002_S4.out out/093/2002.txt
mkdir -p out/093; cp 093-2004P_S11.out out/093/2004P.txt
mkdir -p out/093; cp 093-2012-0_S63.out out/093/2012.txt
mkdir -p out/093; cp 093-2012P_S12.out out/093/2012P.txt
mkdir -p out/093; cp 093-2024-0_S64.out out/093/2024.txt
mkdir -p out/093; cp 093-2024P_S13.out out/093/2024P.txt
mkdir -p out/093; cp 093-2036P_S14.out out/093/2036P.txt
mkdir -p out/102; cp 102-2002-0_S65.out out/102/2002.txt
```



<!-- ### Getting a little more fancy

We're going to jump ahead a bit.  Let's make a simple bash script that will tell us the sizes of all
the FASTA files.  This may be a little daunting, but I aim to show you how to break down a seemingly
complicated problem.  Before going head first into this problem, you need to ask what do you want?
By now you should be familiar with CSV files.  Kinda like a spreadsheet with the cells in the rows
separated by a comma. May I want the output to look like this: 


```csv
Seq1, 2000
Seq2, 50000
Seq3, 500
```

OK, great. We know the end goal.  Now we have to ask ourselves: what does a FASTA file look like?


-->




