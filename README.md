# Cassovary vs. GraphJet

[Cassovary](http://cassovary.io) [1] and [GraphJet](http://graphjet.io/) [2] represent two generations of graph processing engines deployed in production at Twitter. Cassovary was designed as a scale-up in-memory graph processing engine for static graphs, built circa 2010 in Scala. GraphJet was designed as a scale-up in-memory graph processing engine for dynamic graphs, built circa 2014 in Java. Both are open source.

Comparing Cassovary vs. GraphJet is interesting in highlighting the complexity and costs associated with real-time graph processing, since that's the major advance in GraphJet over Cassovary. Note that as a design philosophy, in both cases Twitter has pursued a scale-up approach that avoids graph partitioning. For related discussion, see McSherry et al. [3].

[1] Pankaj Gupta, Ashish Goel, Jimmy Lin, Aneesh Sharma, Dong Wang, and Reza Zadeh. [WTF: The Who to Follow Service at Twitter.](http://dl.acm.org/citation.cfm?id=2488433) Proceedings of the 22th International World Wide Web Conference (WWW 2013), pages 505-514, May 2013, Rio de Janeiro, Brazil.

[2] Aneesh Sharma, Jerry Jiang, Praveen Bommannavar, Brian Larson, and Jimmy Lin. [GraphJet: Real-Time Content Recommendations at Twitter.](http://www.vldb.org/pvldb/vol9/p1281-sharma.pdf) Proceedings of the VLDB Endowment, 9(13):1281-1292, 2016.

[3] Frank McSherry, Michael Isard, and Derek G. Murray. [Scalability! But at what COST?](https://www.usenix.org/conference/hotos15/workshop-program/presentation/mcsherry) Proceedings of the 15th Workshop on Hot Topics in Operating Systems (HotOS XV), 2015.


## Cassovary

Here are instructions for running basic experiments in Cassovary. First, clone the repo:

```
$ git clone git@github.com:twitter/cassovary.git
```

Cassovary has a class [`com.twitter.cassovary.PerformanceBenchmark`](https://github.com/twitter/cassovary/blob/master/cassovary-benchmarks/src/main/scala/com/twitter/cassovary/PerformanceBenchmark.scala) for running performance benchmarks. This class reads graph data from `cassovary-benchmarks/src/main/resources/graphs/`.

Here's how to run global PageRank on graph data that's stored directly in the repo:

```
$ ./sbt "project cassovary-benchmarks" \
    "run-main com.twitter.cassovary.PerformanceBenchmark -local=facebook -globalpr"
$ ./sbt "project cassovary-benchmarks" \
    "run-main com.twitter.cassovary.PerformanceBenchmark -local=wiki -globalpr"
```

Running Cassovary on the [LiveJournal graph from SNAP](https://snap.stanford.edu/data/soc-LiveJournal1.html): First, download the data and put it in the right directory:

```
$ wget http://snap.stanford.edu/data/soc-LiveJournal1.txt.gz
$ cp soc-LiveJournal1.txt.gz cassovary-benchmarks/src/main/resources/graphs/
```

Now we can run the performance benchmark in Cassovary:

```
$ ./sbt "project cassovary-benchmarks" \
    "run-main com.twitter.cassovary.PerformanceBenchmark -gz -out=true -in=false \
       -local=soc-LiveJournal1.txt.gz -separator=9 -globalpr"
```

Note that the performance benchmark by default expects the edge list
to be space delimited. The LiveJournal data is tab separated, which is
why we use the `-separator=9` option (tab is decimal 9).

Use `-h` if we want to see all the options provided by `PerformanceBenchmark`.

## GraphJet

Here are instructions for running basic experiments in GraphJet. First, clone the repo:

```
$ git clone git@github.com:twitter/GraphJet.git
```

Build GraphJet:

```
$ mvn package install -DskipTests
```

The `-DskipTests` option skips tests for the impatient.

GraphJet contains an adapter to read Cassovary graphs. Here's how we would run GraphJet's PageRank implementation on an underlying Cassovary graph:

```
$ mvn exec:java -pl graphjet-demo -Dexec.mainClass=com.twitter.graphjet.demo.PageRankCassovaryDemo \
    -Dexec.args="-inputDir='.' -inputFilePrefix='soc-LiveJournal1.txt.gz' -dumpTopK=10 -threads=1"
```

Use the `-threads` option to set the number of threads.

Note that edges in the LiveJournal graph are sorted by vertex id by default. If we wish to process the graph in a streaming manner, it makes sense to randomly shuffle the edges. Here's how to do it:

```
$ nohup gunzip -c soc-LiveJournal1.txt.gz | \
    awk 'BEGIN{srand();}{print rand()"\t"$0}' | sort -k1 -g | cut -f2- | gzip > \
    soc-LiveJournal1.shuffle1.txt.gz &
```

Now we can run PageRank on a GraphJet graph, after all the edges are read in from the file incrementally (simulating a real-time streaming graph):

```
$ mvn exec:java -pl graphjet-demo -Dexec.mainClass=com.twitter.graphjet.demo.PageRankGraphJetDemo \
    -Dexec.args="-inputFile='soc-LiveJournal1.shuffle1.txt.gz' -dumpTopK=10 -threads=1"
```

Use the `-threads` option to set the number of threads.

Note that in `PageRankCassovaryDemo` we are bulk-ingesting the graph and then running PageRank, whereas with `PageRankGraphJetDemo` we are ingesting each edge incrementally, as in a real-time graph processing scenario. In both cases, however, PageRank is executed on a graph that no longer changes at the end (once the graph is fully ingested).

