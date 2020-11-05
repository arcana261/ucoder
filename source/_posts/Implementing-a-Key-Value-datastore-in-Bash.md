title: Implementing a Key Value datastore in Bash!
author: Mohamad mehdi Kharatizadeh
tags:
  - bash
  - database
  - keyvalue
  - key value
  - cassandra
  - sstable
  - sort
  - look
  - amortized
  - data
  - data engineering
categories:
  - sysadmin
date: 2020-11-04 21:57:00
---
It's been ages since last time I decided to write a post here. But I am finally here :) 

Today, I want to embark you on a funny small project that I suddenly decided to do. Implementing a key-value datastore in bash! using SSTables!

**You can find the link to the reposity [here](https://github.com/arcana261/bashdb).**

## Background

It all started when I was thinking that technically you can throw all spreadsheet tools to the garbage and use bash instead (haha.. A silly thing to do but technically feasible). There are lots of cool tools available at your disposal:

* **sort** which can sort on individual columns or group of columns in ascending or descending order, has inplace sorting and also can make data unique
* **look** which can perform binary search on data to lookup values
* **cut** which can cut columns or remove columns (awk can do that do but cut can be faster for some purposes)
* **awk** which you can essentially write scripts and do lots of calculations on your data.. like taking a SUM or whatever..
* **paste** which can join two seperate CSV or TSV files together
* **tr** which you can use to get rid of extra padding or whitespace in your data. You can also use it to switch from CSV to TSV or whatever
* **jc** which you can use to convert CSV into JSON
* **jq** If you used **jc** to convert CSV into JSON, then **jq** is a powerful tool to perform queries, data transformations, etc. and later save them as either JSON or CSV.

Actually, the latter is a little bit tricky, so as an example, I will show you a code snippet which actually converts CSV to JSON and again from JSON to CSV:

```bash
echo "name,age-mehdi,31-behnaz,32" | tr '-' '\n' | jc --csv | jq -r '([.[0] | to_entries | map(.key)] + (. | map(to_entries | map(.value) ) ) ) as $rows | $rows[] | @csv'
```

In the above example, I used `tr` to convert one liner `echo` output into legitimate CSV by convert dashes into newlines. I then used `jc --csv` to convert CSV data into JSON. And the weird `jq` query converts json into CSV again.

In case you need help, you are welcome in using my bash aliases for this purpose:

```bash
alias trim="tr -s [:blank:] | tr -s [:space:]"
alias bsv2tsv="trim | tr [:blank:] '\t'"
alias csv2json="jc --csv"
alias tsv2csv="tr ',' '-' | tr '\t' ','"
alias bsv2csv="trim | tr ',' '-' | tr [:blank:] ','"
alias tsv2json="tsv2csv | csv2json"
alias bsv2json="bsv2csv | csv2json"
alias json2csv="jq -r '([.[0] | to_entries | map(.key)] + (. | map(to_entries | map(.value) ) ) ) as "'$'"rows | "'$'"rows[] | @csv' | tr -d '"'"'"'"
alias csv2tsv="tr '\t' ' ' | tr ',' '\t'"
alias csv2bsv="tr ' ' '-' | tr ',' ' '"
alias json2tsv="json2csv | csv2tsv"
alias json2bsv="json2csv | csv2bsv"
```

Using above aliases, we can now use `jq` to manipulate data as well as `awk`. Here is a quick example:

```bash
echo "name,age-behnaz,32-mehdi,31" | tr '-' '\n' | csv2json | jq -M 'sort_by(.age)' | json2csv
```

In the above example, we have used `sort_by` filter to perform sorting. But literally all the power that comes with `jq` can be exploited this way!

## Architecture

At the core, we want to implement SSTables. Now, an SSTable is simply a fancy word for a sorted list. But this is essentially how many key-value databases work at core. There is a good [blog post](http://distributeddatastore.blogspot.com/2013/08/cassandra-sstable-storage-format.html) explaining SSTables which I strongly recommend reading it if your are not familiar yet.

In a nutshell, we want to store data in a sorted fashion on disk so we can perform binary search in `O(lg n)` when looking up values. But maintaining a sorted array in every insert is `O(n)` so we have to break our sstable into multiple lists as we go along the way. However, after some time, we would be stuck with a lot of sstables which have duplicate data that overrides one another. So we would have to compact, or merge, these sstables together. So we have these operations on our datastore:

* **Insert**: This adds a new item to the latest sstable. Since sstables are bounded by a constant size, say `B` before we split them, the cost of this operation is `O(B)`.
* **Split**: This happens before inserting an item, when we figure out that there are `B` items in our latest sstable and we decide to write our insert to a new sstable
* **Compact**: This operation merges lots of SSTables into one unified compact sstable with all the duplicate data removed. Suppose we have maximum of `S` sstables before we decide to compact. The time complexity of this operation is `O(Sn)` in which `n` is number of items in our data store.

So, given all that, what's the time complexity of a single insertion to our data store? Given parameters `n`, `B` and `S`:

* We would create `ceil(n / B)` sstables each with a maximum size of `B`
* We would perform `floor( ceil(n/B) / S )` compact operations, each of time complexity `O(n + BS^2)`. Because we have `O(BS)` items for every compact operation done and every item can be counted `O(S)` times, Plus we have a big sstable which all items merge into.
* Therefore, total time spent in `n` insert operations is `O((n^2)/BS + nS + nB)`. `nB` comes from the fact that maintaining the latest SSTable with regard to a insert operation is `O(B)`.
* Therefore, the amortized time complexity of every insert operation is `O((n/BS) + S + B)`.
* Cost of lookup operation is `O(S lg B)`

## How it's done

To show you how the implementation is done, I would break it down to how I've done insert operation, compact operation, remove operation and lookup operation.

### Insert Operation

For an insert operation, we use the sort command:

```bash
    sort -m -u -t '|' -k 1,1 $new_file $sstable > $temp_file
```

In this command:


1. `-m` merges two sorted files instead of soring them
2. `-u` makes result unique
3. `-t '|'` is just our field seperator (column seperator) in the SSTable file
4. `-k 1,1` tells sort to just use first column

### Compact Operation

Using command sort we maintain and also compact (merge) multiple SSTables together. The syntax is the same as Insert Operation.

### Lookup Operation

Also for lookups, we perform the following:

```bash
    look -b -t '|' $key $sstable | cut -d '|' -f 2
```

In this command:

1. `-b` performs a binary search of O(lg n)
2. `-t '|'` is just our field seperator (column seperator) in the SSTable file
3. `cut -d '|' -f 2` simply selects second column and prints it

### Remove Operation

For a remove operation, we simply insert a marker, `<REMOVED>` to the list.

## Any Thoughts?

I don't know of real life uses of having SSTables implemented in bash. I just wanted to do this for fun and for learning bash. If you have any suggestion/improvements or you have a real life usage for this.. Feel free to leave a comment or make an MR.

Have a nice day!