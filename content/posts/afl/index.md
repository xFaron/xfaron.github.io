---
title: "Unveiling AFL (American Fuzzy Lop): My reading notes"
date: 2025-07-07
tags: ["fuzzing"]
description: "My reading notes on AFL"
draft: false
---

{{< lead >}}
Note : The content below are my reading notes on AFL. They are accurate to the best of my beliefs. But, if you find any errors, let me know : )
{{< /lead >}}
{{< katex >}}

American Fuzzy Lop (AFL) is a groundbreaking fuzzing tool that revolutionized coverage-guided greybox fuzzing. Its design philosophy emphasizes performance, simplicity, and deep system understanding. The below notes explores some of the most critical mechanics and design principles behind AFL.

## Coverage measurements

There are two forms of coverage measurements
- Coarse Hit count registration 
- Storing Tuples of branches explored

Each branch of the program is associated with a unique id. Now, based on this, we can create tuples of the form (branch1_id, branch2_id). Such ordered sequence of tuples, if follow an execution trace, can be considered to be a path.

Usually, some lightweight instrumentation is injected into the code at compile time which looks something like this

```
cur_location = <COMPILE_TIME_RANDOM>;
shared_mem[cur_location ^ prev_location]++; 
prev_location = cur_location >> 1;
```

The `shared_mem` is a 64KB bitmap in the case of AFL, and it stores all the unique tuple counts explored till then.

Basically, `cur_location` is the identifier for a given branch, `cur_location ^ prev_location` can be considered a [^1]semi unique id for the tuple at that instant.

Suppose the program execution trace is as follows $$A\rightarrow B\rightarrow C\rightarrow D\rightarrow E$$
Then, the `shared_mem` might store something which intuitively looks like what is given below

```
shared_mem = {
	(A,B) : 1
	(B,C) : 1
	(C,D) : 1
	(D,E) : 1
}
```


> [!NOTE] For intuitiveness only!
> The above map is just an intuitive explanation. In the actual scenario, `cur_location ^ prev_location`  is the key, which comes out to be some index in the byte-array, and the value is generally the hitcounts

Another example, say $$A\rightarrow B\rightarrow C\rightarrow A\rightarrow B$$
comes out with a `shared_mem` as shown below

```
shared_mem = {
	(A,B) : 2
	(B,C) : 1
	(C,A) : 1
}
```

There is a specific importance to `prev_location = cur_location >> 1`. This help the program differentiate between tuples like $(A,B)$ and $(B,A)$. It also help differentiate $(A,A)$ and $(B,B)$ ( When two same numbers are XOR'ed, they return 0 )


> [!NOTE] From Paper : Coverage-based Greybox Fuzzing as Markov Chain
> The variable cur_location identifies the current basic block. Its random identifier is generated at compile time. Variable shared_mem[] is a 64 kB shared memory region. Every byte that is set in the array marks a hit for a particular tuple (A, B) in the instrumented code where basic block B is executed after basic block A. The shift operation in Line 3 preserves the directionality [(A, B) versus (B, A)]. A hash over shared_mem[] is used as the path identifier.


## Detecting new behaviour

The fuzzer maintains a global map of tuples seen in previous executions; this data can be rapidly compared with individual traces and updated in just a couple of dword- or qword-wide instructions and a simple loop.

Now, there are two ways to flag a seed to produce [^2]interesting output

### Based on Coarse Hit counts

Basically, this means that, if the hit count number for a particular tuple for a particular seed moves the hit count into a new bucket, then it regarded as interesting.

AFL uses the following hit count buckets given below $$1,2,3,4-7,8-15,16-31,32-127,128+$$
This buckets are obviously chosen for their binary significance. ( Faster operations to check if a number is in these buckets ). Another significance is that, most of time, empirically, a jump from 1 to 2 hit counts can be considered to be significant, from say jump from 47 to 48 hit counts. This also helps avoid the path explosion problem.

Now, to avoid confusion, look at the examples from the previous section. For the tuple $(A,B)$, the number of hit counts falls into a new bucket which has not been touched until then. Due to this factor, the second seed is considered interesting
### Based on new tuples

This is quite literally what the title says. If the program execution covers new branches, or an old branch in some new order ($A\rightarrow B$ might have become $B\rightarrow A$) , then that seed is also considered interesting.

## Evolution of seeds

All the seeds are stored in a queue. When a new interesting seed is found, it is added to the queue to be processed.

> [!NOTE] Power schedule
> There seems to be no concept of power schedules at this point in the making of AFL

Mutation strategies are applied. According to the author, mutation strategies lead to better results compared to genetic algorithms. Intuitively, it looks like random mutations evenly explores the search space, finding more interesting executions, rather than a genetic one, which might just go exploring a certain section of the code.
## Aggressive timeouts

This is also a feature of AFL such that if an input takes longer than some specified limit (in paper, said 5 times the initial run time for the first seed modulo 20 ms), then, that input is no longer processed. This is to keep the fuzzer moving and to be not boggled by slow inputs. The author of AFL also hopes that the fuzzer finds some other input which can uncover the same path that a slow input might have uncovered.

## Reduction of seeds

This favoured list of seeds after a certain point ( when? ) is reduced into a smaller corpus of seeds that has the same code coverage without all the seeds. It reduces redundancy. Usually, this corpus is seen to be 5 to 10 times smaller

> [!NOTE] AFL Authors notes
> To optimize the fuzzing effort, AFL periodically re-evaluates the queue using a fast algorithm that selects a smaller subset of test cases that still cover every tuple seen so far, and whose characteristics make them particularly favorable to the tool
> 
> The algorithm works by assigning every queue entry a score proportional to its execution latency and file size; and then selecting lowest-scoring candidates for each tuple.
> 
> The tuples are then processed sequentially using a simple workflow:
> 1) Find next tuple not yet in the temporary working set
> 2) Locate the winning queue entry for this tuple
> 3) Register *all* tuples present in that entry's trace in the working set
> 4) Go to #1 if there are any missing tuples in the set.
>
> The generated corpus of "favored" entries is usually 5-10x smaller than the starting data set. Non-favored entries are not discarded, but they are skipped with varying probabilities when encountered in the queue:
> - If there are new, yet-to-be-fuzzed favorites present in the queue, 99% of non-favored entries will be skipped to get to the favored ones.
> - If there are no new favorites:
> 	- If the current non-favored entry was fuzzed before, it will be skipped 95% of the time.
> 	- If it hasn't gone through any fuzzing rounds yet, the odds of skipping drop down to 75%.
> 
> Based on empirical testing, this provides a reasonable balance between queue cycling speed and test case diversity.

So, basically, what I got is, there is a global hashmap that stores all the tuples to their hit count buckets

Each new reduction process, the whole queue to inputs is evaluated. They are assigned a score using the following formula $$Score = k\times Time\times InputSize$$Lower score implies higher preference here. Based on these score, the keys of the hash map are iterated, and for each tuple, it goes through the corpus in ascending order to find the best seed for that tuple.

All these tuple to seed pairs are stores, and then the algorithm is executed

I believe from this, I understand that there might be something like a favoured metric in each seed. Also, a metric that states how many times it has been executed ( although not entirely necessary now that I think about it ). Based on these info, their energy may have been reevaluated by a power schedule

So, in summary, the power schedule assignEnergy function is run, which follows the above steps.
## Reductions of size of input files / seeds

There are specific mechanisms in place that reduce the file size of these input seeds, while trying to preserve all the data associated with these seeds ( bugs, coverage etc ). Improves data management, and explosion of memory.

The actual minimization algorithm in `afl-tmin`:

  1) Attempt to zero large blocks of data with large stepovers. Empirically, this is shown to reduce the number of execs by preempting finer-grained efforts later on.

  2) Perform a block deletion pass with decreasing block sizes and stepovers,binary-search-style. 

  3) Perform alphabet normalization by counting unique characters and trying to bulk-replace each with a zero value.

  4) As a last result, perform byte-by-byte normalization on non-zero bytes.

## Mutation strategy

AFL starts by mutating its corpus of seeds in a *deterministic* way. According to the authors notes, they use 3 mutations in this stage

1. Sequential bit flips ( varying length and stepovers )
2. Sequential addition and subtraction of small integers
3. Sequential insertion of known interesting integers (0, 1, INT_MAX etc)

After this stage, AFL moves into its *undeterministic* stage, where it follows the following mutations

1. Stacked bit flips
2. Insertions
3. Deletions
4. Arithmetic's
5. Splicing

Chiefly due to performance, simplicity and reliability concerns, AFL does not try to reason program states with mutations

But, there is one exception to the above case. When in the stage of deterministic fuzzing, if the test case does not yield a new checksum for its path, then it directly moves to random mutations. This has been seen to improve the performance of the fuzzer.

[^3]Effector maps also seems to be implemented, although, I am not sure. By external research, effector maps direct mutation to bytes that matter. Suppose for an input, only the first 100 bytes matter out of the total 1024, then mutating 101 to 1024 is a waste. To avoid this, effector maps is used.

## Dictionaries

In essence, when basic, typically easily-obtained syntax tokens are combined together in a purely random manner, the instrumentation and the evolutionary design of the queue together provide a feedback mechanism to differentiate between meaningless mutations and ones that trigger new behaviors in the instrumented code - and to incrementally build more complex syntax on top of this discovery.

The dictionaries have been shown to enable the fuzzer to rapidly reconstruct the grammar of highly verbose and complex languages such as JavaScript, SQL, or XML; several examples of generated SQL statements are given in the blog post mentioned above.

Interestingly, the AFL instrumentation also allows the fuzzer to automatically isolate syntax tokens already present in an input file. It can do so by looking for run of bytes that, when flipped, produce a consistent change to the program's execution path; this is suggestive of an underlying atomic comparison to a predefined value baked into the code. The fuzzer relies on this signal to build compact "auto dictionaries" that are then used in conjunction with other fuzzing strategies.

## Unique crash registration

The solution implemented in afl-fuzz considers a crash unique if any of two conditions are met:

  - The crash trace includes a tuple not seen in any of the previous crashes,

  - The crash trace is missing a tuple that was always present in earlier
    faults.

The approach is vulnerable to some path count inflation early on, but exhibits a very strong self-limiting effect, similar to the execution path analysis logic that is the cornerstone of afl-fuzz.

## Fork server

Basically, all the initialisation is done for the program ( linking, importing libraries etc. ), and then the program is stalled at the input phase. Then, this process is copied. This reduces a lot of repeated overhead of compilation procedures. 

---

So, thats to conclude some features of AFL. I wrote some more notes on AFL, but, I think it would be wiser to read it from the actual paper itself, because, that explains AFL really well.

> Coverage-based Greybox Fuzzing as Markov Chain : https://mboehme.github.io/paper/TSE18.pdf#page=2.98

The above paper talks about improving AFL with power schedules, which are a whole another topic, which I won't be discussing here, since, the whole matter of Fuzzing has been discussed in detail in the "Fuzzing Book[^4]". But, back to the point, under Section 2.1, they have discussed beautifully about AFL. Go check that out

PS : There are issues with some math typesetting. Will fix at the earliest!

---

AFL’s strength lies in its pragmatic and efficient design choices. Rather than complex symbolic reasoning, it embraces lightweight, empirically validated heuristics that have been shown to work exceptionally well in practice. Kinda cool how very simple and naive thinking to testing is among the best work seen in this feild.



[^1]: These id's are not unique to be precise for large programs in general. These id's depend on the compile time random number. At the same time, due to the size of the bitmap being 64KB, a total of $64\times1024$ unique tuples can be stored. So, usually, for huge programs, tuples overlap. But, this is still preferred over creating unique id's for each tuple due to the marginal speed gains in using a 64KB mem ( which conveniently fits into an L2 cache )

[^2]: Although not clear from https://github.com/google/AFL/blob/master/docs/technical_details.txt due to his example, I am assuming that both new hit counts and new tuples make a seed intersting

[^3]: Refer to https://github.com/google/AFL/blob/master/docs/technical_details.txt for more details, under section 6, last paragraph.

[^4]: By Andreas Zeller, Rahul Gopinath, Marcel Böhme, Gordon Fraser, and Christian Holler. Refer https://www.fuzzingbook.org/
