# Cassovary vs. GraphJet

## Cassovary

Clone Cassovary:

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

Running Cassovary on the [LiveJournal graph from SNAP](https://snap.stanford.edu/data/soc-LiveJournal1.html) - download the data and put it in the right directory:

```
$ wget http://snap.stanford.edu/data/soc-LiveJournal1.txt.gz
$ cp soc-LiveJournal1.txt.gz cassovary-benchmarks/src/main/resources/graphs/
```

Now you can you run the performance benchmark in Cassovary:

```
$ ./sbt "project cassovary-benchmarks" \
    "run-main com.twitter.cassovary.PerformanceBenchmark -gz -out=true -in=false \
       -local=soc-LiveJournal1.txt.gz -separator=9 -globalpr"
```

Note that the performance benchmark by default expects the edge list
to be space delimited. The LiveJournal data is tab separated, which is
why we use the `-separator=9` option (tab is decimal 9).

Use `-h` if you want to see all the options provided by `PerformanceBenchmark`.

## GraphJet

If the raw graph edges are sorted by vertex id, it might be a good
idea to shuffle them randomly. 

```
$ nohup gunzip -c soc-LiveJournal1.txt.gz | \
    awk 'BEGIN{srand();}{print rand()"\t"$0}' | sort -k1 -n | cut -f2- | gzip > \
    soc-LiveJournal1.shuffle1.txt.gz &
```

See [this thread](http://stackoverflow.com/questions/2153882/how-can-i-shuffle-the-lines-of-a-text-file-on-the-unix-command-line-or-in-a-shel) for additional details and discussion.

Running PageRank on GraphJet:
1. Install and cd into GraphJet
```
git clone -b dev https://github.com/yb1/GraphJet.git GraphJet_pagerank
cd GraphJet_pagerank
```
2. (Optional) increase the memory if the graph is large. (Add configuration in plugin in pom.xml)
```
<plugin>
  ...
  <configuration>
      <argLine>-Xmx100000m</argLine>
  </configuration>
  ...
</plugin>
```
3. Compile and run
```
 mvn package install -DskipTests
 mvn exec:java -pl graphjet-demo -Dexec.mainClass=com.twitter.graphjet.demo.PageRankDemo -Dexec.args="-inputFile='path-to-file-containg-graph'"
```
Also, other arguments can be set in -Dexec.args. (e.g "-inputFile='path-to-file-containg-graph' -maxEdgesPerSegment=10000")

See also McSherry et al.'s ["Scalability! But at what COST?"](https://www.usenix.org/conference/hotos15/workshop-program/presentation/mcsherry) paper and [associated code on GitHub](https://github.com/frankmcsherry/COST).

