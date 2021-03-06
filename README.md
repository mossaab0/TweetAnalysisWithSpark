# Tweet Analysis with Spark

This tutorial shows how to use Spark to analyze crawls from Twitter's streaming API, which is formatted as a sequence of JSON objects, one per line. We assume CDH 5.3.0.

Fire up the Spark shell:

```
$ export HADOOP_CONF_DIR=/etc/hadoop/conf
$ spark-shell --master yarn-client --jars gson-2.3.1.jar --num-executors 10
```

The need for setting up the environment variable is due to [this issue](https://issues.cloudera.org/browse/DISTRO-664). You can download the jar [here](http://search.maven.org/#artifactdetails%7Ccom.google.code.gson%7Cgson%7C2.3.1%7Cjar).

If you want to play with individual tweets:

```
import com.google.gson._

val jsonParser = new JsonParser()
val gson = new GsonBuilder().setPrettyPrinting().create()

val raw = sc.textFile("/path/to/file");

// Take five JSON records and print each out.
for (r <- raw.take(5)) {
  println(gson.toJson(jsonParser.parse(r)))
}
```

Here's the [Gson API javadoc](http://google-gson.googlecode.com/svn/trunk/gson/docs/javadocs/index.html).

Now let's analyze the tweets at scale. Here's how we start:

```
import com.google.gson._

val raw = sc.textFile("/path/to/file");

var tweets =
  raw.map(line => {
    val jsonParser = new JsonParser()
    val obj = jsonParser.parse(line).getAsJsonObject()
    if (obj.has("delete")) {
      null
    } else {
      (obj.get("id").getAsLong(), obj.get("text").getAsString())
    }
  }).filter(x => x != null)
```

The variable `tweets` now holds an RDD of (tweetid, tweet text) pairs, with deletes filtered out.

Let's continue with word count:

```
val wc =
  tweets.flatMap(t => t._2.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
```

Let's say, find common terms, sort them, and save output to disk:

```
wc.filter(t => t._2 > 100).sortByKey().saveAsTextFile("wc_out")
```

Say that you have a method `extractFeatures` that assigns a `Float` value to every `String` feature:

```
def extractFeatures(jsonTweet: String) : Map[String, Float] = {
    new HashMap[String, Float]
  }
```

The following builds a dictionary of features that appear at least in `countThreshlohd` documents, including the information needed to normalize the distribution of values into the normal standard distribution.

```
def buildDictionary(training: String, dictionary: String, countThreshlohd: Int) {
    sc.textFile(training).
      flatMap(jsonTweet => asScalaSet(extractFeatures(jsonTweet).entrySet).seq).
      map(a => (a.getKey, (a.getValue, 0d, 1))).
      reduceByKey((a, b) => {
        var delta = (a._1 / a._3) - (b._1 / b._3)
        var weight = (a._3 * b._3).toDouble / (a._3 + b._3)
        var sum = a._1 + b._1
        var diff = a._2 + b._2 + delta * delta * weight
        var count = a._3 + b._3
        (sum, diff, count)
      }).
      filter(a => a._2._3 >= countThreshlohd).
      map(a => a._1 + "\t" + a._2._1 / a._2._3 + "\t" + Math.sqrt(a._2._2 / a._2._3)).
      saveAsTextFile(dictionary)
  }
```

Next we need to load the vocabulary in the application driver, and then send that vocabulary to the executers, which will save the output.

```
  def convertToLibsvm(dic: Map[String, (Float, Float, Int)], input: String, output: String) {
    sc.textFile(input).
      map(jsonTweet => extractFeatures(jsonTweet).entrySet).

      map(a => (a.filter(kv => dic.contains(kv.getKey)).
        map(kv => {
          var triple = dic.get(kv.getKey)
          var mu = triple._1
          var sd = triple._2
          var feature = triple._3
          var value = kv.getValue
          if (value != 0) {
            value = if (sd == 0) 1 else (value - mu) / sd
          }
          (feature, value)
        }).filter(a => a._2 != 0).toArray.sorted.
        map(kv => kv._1 + ":" + kv._2).mkString(" "))).
      saveAsTextFile(output)
  }

def loadDictionaryAndconvertToLibsvm(dictionary: String, inputTrain: String, outputTrain: String, inputDev: String, outputDev: String, inputTest: String, outputTest: String) {
    var v = sc.textFile(dictionary).map(line => line.split("\t")).
      map(triple => (triple(0), (triple(1).toFloat, triple(2).toFloat, 0))).collect

    for (i <- 0 to v.size - 1) {
      v(i) = (v(i)._1, (v(i)._2._1, v(i)._2._2, i + 1))
    }

    var dic = v.toMap

    convertToLibsvm(dic, inputTrain, outputTrain)
    convertToLibsvm(dic, inputDev, outputDev)
    convertToLibsvm(dic, inputTest, outputTest)
  }
```
