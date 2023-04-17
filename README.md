## Running Spark on MAC OS-X

Shows how to install and work with the fundamentals of Apache Spark. The working example throughout will be the "Hello World" of Big Data, which is a word count.

### Installation

Installation is straight forward with homebrew.

If you need java you can run:

```
# Check Java version if already installed
$ java --version
# Run below command to install Java8
$ brew cask install java8
# Latest java version as of Aug-2021
$ brew info java
openjdk: stable 16.0.2 (bottled) [keg-only]
# Run below command to install latest java
$ brew cask install java
```

I already have Java installed so I just needed to run :

```
$ brew install scala
$ brew install apache-spark
$ brew install sbt # this is used for builds
```

### Using the repl

Now the spark shell will be available to you. Just type `spark-shell` in the command line and a Scala repl will kick off. In the shell, two variables will automatically be created, `sc` and `spark`. From here you will be able to run a local spark job on your machine in the repl.

- `sc` represents the "spark context", the main starting point of a spark application. This is the delegator, on distributed systems this will actually control the workers across your distributed system
- `spark` - used for spark SQL library. Here's some sample code you can run in the repl to produce a word count of this `README.md` file! First, `cd` into this repo and start the repl with `spark-shell`, then type each line:

```
scala > val textFile = sc.textFile("README.md")
scala > val tokenizedFileData = textFile.flatMap(line=>line.split(" "))
scala > val countPrep = tokenizedFileData.map(word=>(word, 1))
scala > val counts = countPrep.reduceByKey((accum, x) => accum + x)
scala > val sortedCount = counts.sortBy(kvPair=>kvPair._2, false)
scale > sortedCount.saveAsTextFile("/path/to/output/readmeWordCount")
```

Note: None of the transforms written here will actually be executed on the data until you perform the write command at the end.

The output from this code will be a directory containing partitioned data. Looking something like this:

```
wordcount
├── _SUCCESS
├── part-00000
└── part-00001
```

There will be `n` number of partitioned files containing the word counts distributed across them with the largest counts appearing first.

### To build Spark a project

Let's now move beyond using the `repl` to creating a `.scala` app, using a build tool to create `.jar` file. Then executing the jar file to produce the same result we produced from the repl.

1. Create a project directory that looks like this:

   ```
   ├── src
   │   └── main
   │       └── scala
   │           └── WordCounter.scala
   └── build.sbt
   ```

   Inside of `WordCounter.scala` you will need the following:

   ```
   /* WordCounter.scala */
   package main

   import org.apache.spark.SparkContext
   import org.apache.spark.SparkConf

   object WordCounter {
     def main(args: Array[String]) {
       /* code to execute goes here */
     }
   }
   ```

   Spark will automatically execute the code contained in this main function. You will need to define the variable `sc` yourself now, since before the repl did it for us. So it should look something like this:

   ```
    /* WordCounter.scala */
   package main

   import org.apache.spark.SparkContext
   import org.apache.spark.SparkConf

   object WordCounter {
     def main(args: Array[String]) {
       val conf = new SparkConf().setAppName("Word Counter")
       val sc = new SparkContext(conf)
       val textFile = sc.textFile("/path/to/readme/README.md")
       val tokenizedFileData = textFile.flatMap(line => line.split(" "))
       val countPrep = tokenizedFileData.map(word => (word, 1))
       val counts = countPrep.reduceByKey((a,x)=> a+x)
       val sortedCounts = counts.sortBy(kvPair => kvPair._2, false)
       sortedCounts.saveAsTextFile("/path/to/output/readmeWordCount")
     }
   }
   ```

2. Next we need to include an `sbt` configuration file: `build.sbt`

   ```
   name := "Word Counter"

   version := "1.0"

   scalaVersion := "2.12.17"

   libraryDependencies += "org.apache.spark" %% "spark-core" % "3.4.0"
   ```

3. Now you can create the `.jar` file by running:

   ```
   $ sbt package
   ```

   You should now see a created jar file in a location similar to `target/scala-2.12/word-counter_2.12-1.0.jar`

4. We can now execute the `.jar` file with another built in utility from spark called `spark-submit`:

   ```
   $ spark-submit --class "main.WordCounter" --master "local[*]" /path/to/jar/word-counter_2.12-1.0.jar
   ```

   Note: For now we are still just using our local machine for execution. This is specified with `--master "local[*]"`, the \* expresses that as many cors that are available will be used rather than explicitly specifying.

   You should now see the same output we created from the repl.
