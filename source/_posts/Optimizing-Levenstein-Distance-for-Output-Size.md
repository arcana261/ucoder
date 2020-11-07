title: Optimizing Levenstein Distance for Output Size
author: Mohamadmehdi Kharatizadeh
tags:
  - dp
  - Dynamic Programming
  - C++
  - Levenstein
  - Distance
categories:
  - Algorithm
date: 2020-11-06 22:05:00
---
We are all familiar with [Levenstein Distance](https://en.m.wikipedia.org/wiki/Levenshtein_distance). A classic dynamic programming algorithm designed to find the minimum number of operations necessary for converting a text into another text.

So... What if we were to encode that *difference* in a file for example. And also suppose that we want to minimize the size of this differential file. Does the levenstein distance help us as it is?

**NO**

The answer is pure **NO**.

Why? It is simple to see that for *INSERT* operation, we have to encoded inserted characters in the file, plus a command character (we'll get to that) while *DELETE* operation has only the command character.

But I'm getting ahead of myself! Let's list items that we need to look into:

1. Design differential file format
2. Tackle large domain space
3. Modify Levenstein distance algorithm

## Design differential file format

In this section, we'll design a file format to use to encode difference between two texts.

We need two primary components:

1. A command character, which shows what operation we are expecting in order to build the second text from the difference. Valid operations are **I** (Insert Operation), **K** (Keep Operation), **D** (Delete Operation), **R** (Replace Operation). I will discuss the meaning of these operations shortly.
2. A length operator. We can further squeeze the output size if we used variable length binary integers in our file format. At least *Google Protobuf* and therefore *gRPC* are famous libraries that I know of that use the same semantics of variable length binary integer size. Also UTF-8 uses the same mechanism for it's encoding.

So let me further explain the command operator. Suppose we want to reach text **B** from text **A**. In the following statements, *n* is encoded using a length operator.

* **Insert** command, inserts *n* characters from text **B**.
* **Delete** command, removes *n* characters from text **A**.
* **Keep** command presumes following *n* characters are the same between text **A** and text **B**.
* **Replace** command replaces next *n* characters of text **A** from text **B**

We can also encode variable length integers by encoding integers into segments of 7 bits. Here is how it works:

1. If integer *I* is between `[0, 0x7F]` write it as it is.
2. If integer *I* is between `[0x80, 0x7FFF]`, then write the first 7 bits bitwise ORed `0x80` and next 7 bits in the second byte.
3. The algorithm continues by marking 8th bit of every output byte for every 7 bits of input data.

I think this would be easier to understand if I present some code first:

```cpp
void write_size(size_t size) {
	if (size <= 0x7F) {
    	uint8_t x = (uint8_t)size;
        cout.write((char*)&x, 1);
    } else {
    	uint8_t x = (uint8_t)((size & 0x7F) | 0x80);
        cout.write((char*)&x, 1);
        write_size(size >> 7);
    }
}
```

So to wrap up, I use a grammer to describe my differential file format of choice:

```
DifferentialFile := Command*
Command := (Insert | Delete | Keep | Replace)
Insert := N Text
Delete := N
Keep := N
Replace := N Text
N := <variable length binary integer>
Text := N character long string
```

## Tackle Large Problem Space

The time complexity of classic levenstein distance is `O(n m)`. This is not good for handling large files. Specially since `O(n m)` memory is also needed. We need a little more engineering and less science to tackle this issue.

What I've found to be useful, was to break large files into smaller *chunks*. The size of the *chunk* somehow resembles *compression ratio* in our case. The larger the *chunk size* the better compression we would have!

Despite using chunks, In order to increase efficiency, we need a greedy algorithm to merge differential output result of consecutive chunks together. For example, if we have `Keep 20` for a chunk and `Keep 10` for it's next chunk, instead of saying `Keep 20 Keep 10` we can just say `Keep 30`.

The greedy algorithm that I invented and found useful is as following:

1. Merge length of consecutive similar commands
2. Replace consecutive (Insert,Delete) or (Delete,Insert) commands of same length with Replace command.
3. Repeat above until there is no more change

The above can be easily turned into pseudocode below:

```
next_commands := commands
commands := []
while next_commands <> commands; do 
	commands := next_commands
    next_commands := []
	
	for i := 1 to /commands/; do
    	if commands[i - 1].cmd = commands[i].cmd; then
        	next_commands.append(command(commands[i].cmd, commands[i-1].n + commands[i].n))
        elif commands[i-1].n = commands[i].n and ((commands[i].cmd = INSERT and commands[i-1].cmd = DELETE) or (commands[i].cmd = DELETE and commands[i-1].cmd = INSERT)); then
        	next_commands.append(command(REPLACE, commands[i].n))
        else
        	next_commands.append(commads[i-1])
            next_commands.append(commands[i])
        fi
    done
done
```

What is the time complexity? For chunks of size `B` we could have `O(B)` commands so the time complexity of above greedy algorithm can be as much as `O(B^2)`. This is still manageable, since classic levenstein distance can not be run faster than `O(B^2)`.

Please remember that I have no formal proof to why above greedy algorithm gives the optimum result nor do I really intend to optimize merger of chunks. This is just a bit of engineering added to our methodology, nothing more!

## Modify Levenstein Distance Algorithm

It is easy to notice that the cost of command we intend to undertake at subproblem `[i,j]`is dependent on current command of parent subproblem. Why? Because the parent subproblem has paid the cost of both command operator and length operator.

Now, a little bit of engineering to consideration of length operator again, is that we discard length operator as a whole in our algorithm.

1. If chunk size is always less than 128, our algorithm remains theoretically correct
2. If chunk size is always less than 0x7FFF, consideration of length operator does not change time complexity, but it will have a huge impact on run-time statistics of our algorithm.
3. I don't think chunk size would ever go beyond 0x7FFF in real world. Huge amount of resources is required to do so and it is infeasible.

So having removed the length operator from problem space, we formulate the problem space as following dimentions:

1. **i** index on text **A**
2. **j** index on text **B**
3. **C** command of parent problem space

Now, calculation of every `[i, j, C]` is as same as `[i, j]` of classical levenstein distance algorithm with the exception that:

Suppose we choose command C2:
1. If `C2 = C` then calculate as we had done in classical levenstein distance
2. If `C2 <> C` then add cost 2 (1 for command operator and 1 for a supposedly 1-byte length operator) plus calculation done in classical levenstein distance.

## Result

You can find a crappy code [here](https://github.com/arcana261/notes/blob/master/sysconfig/bash/capture/leven.cxx) as I have implemented the above algorithm in C++.