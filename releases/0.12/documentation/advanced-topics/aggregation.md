---
layout: default
title: ReactiveMongo 0.12 - Aggregation Framework
---

## Aggregation Framework

The [MongoDB Aggregation Framework](http://docs.mongodb.org/manual/reference/operator/aggregation/) is available through ReactiveMongo.

### ZipCodes example ###

Considering there is a `zipcodes` collection in a MongoDB, with the following documents.

{% highlight javascript %}
[
  { '_id': "10280", 'city': "NEW YORK", 'state': "NY",
    'population': 19746227, 'location': {'lon':-74.016323, 'lat':40.710537} },
  { '_id': "72000", 'city': "LE MANS", 'state': "FR", 
    'population': 148169, 'location': {'long':48.0077, 'lat':0.1984}},
  { '_id': "JP-13", 'city': "TOKYO", 'state': "JP", 
    'population': 13185502L, 'location': {'lon':35.683333, 'lat':139.683333} },
  { '_id': "AO", 'city': "AOGASHIMA", 'state': "JP",
    'population': 200, 'location': {'lon':32.457, 'lat':139.767} }
]
{% endhighlight %}

**Distinct state**

The [`distinct`](https://docs.mongodb.org/manual/reference/command/distinct/) command, to find the distinct values for a specified field across a single collection.

In the MongoDB shell, such command can be used to find the distinct states from the `zipcodes` collection, with results `"NY"`, `"FR"`, and `"JP"`.

{% highlight javascript %}
db.runCommand({ distinct: "state" })
{% endhighlight %}

Using the ReactiveMongo API, it can be done with the corresponding [collection operation](../../api/index.html#reactivemongo.api.collections.GenericCollection@distinct[T]%28key:String,selector:Option[GenericCollection.this.pack.Document],readConcern:reactivemongo.api.ReadConcern%29%28implicitreader:GenericCollection.this.pack.NarrowValueReader[T],implicitec:scala.concurrent.ExecutionContext%29:scala.concurrent.Future[scala.collection.immutable.ListSet[T]]).

{% highlight scala %}
import scala.concurrent.{ ExecutionContext, Future }

import reactivemongo.bson.BSONDocument
import reactivemongo.api.collections.bson.BSONCollection

def distinctStates(col: BSONCollection)(implicit ec: ExecutionContext): Future[Set[String]] = col.distinct[String, Set]("state")
{% endhighlight %}

**States with population above 10000000**

It's possible to determine the states for which the sum of the population of the cities is above 10000000, by [grouping the documents](http://docs.mongodb.org/manual/reference/operator/aggregation/group/#pipe._S_group) by their state, then for each [group calculating the sum](http://docs.mongodb.org/manual/reference/operator/aggregation/sum/#grp._S_sum) of the population values, and finally get only the grouped documents whose population sum [matches the filter](http://docs.mongodb.org/manual/reference/operator/aggregation/match/#pipe._S_match) "above 10000000".

In the MongoDB shell, such aggregation is written as bellow (see the [example](http://docs.mongodb.org/manual/tutorial/aggregation-zip-code-data-set/#return-states-with-populations-above-10-million)).

{% highlight javascript %}
db.zipcodes.aggregate([
   { $group: { _id: "$state", totalPop: { $sum: "$pop" } } },
   { $match: { totalPop: { $gte: 10000000 } } }
])
{% endhighlight %}

With ReactiveMongo, it can be done as using the [`.aggregate` operation](../../api/index.html#reactivemongo.api.collections.GenericCollection@aggregate%28firstOperator:GenericCollection.this.PipelineOperator,otherOperators:List[GenericCollection.this.PipelineOperator],explain:Boolean,allowDiskUse:Boolean,cursor:Option[GenericCollection.this.BatchCommands.AggregationFramework.Cursor]%29%28implicitec:scala.concurrent.ExecutionContext%29:scala.concurrent.Future[GenericCollection.this.BatchCommands.AggregationFramework.AggregationResult]).

{% highlight scala %}
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

import reactivemongo.bson.{ BSONDocument, BSONString }
import reactivemongo.api.collections.bson.BSONCollection

def populatedStates(col: BSONCollection): Future[List[BSONDocument]] = {
  import col.BatchCommands.AggregationFramework.{
    AggregationResult, Group, Match, SumField
  }

  val res: Future[AggregationResult] = col.aggregate(
    Group(BSONString("$state"))( "totalPop" -> SumField("population")),
    List(Match(BSONDocument("totalPop" -> BSONDocument("$gte" -> 10000000L)))))

  res.map(_.documents)
}
{% endhighlight %}

> The local `import col.BatchCommands.AggregationFramework._` is required, and cannot be replaced by a global static `import reactivemongo.api.collections.BSONCollection.BatchCommands.AggregationFramework._`.
> The type `.BatchCommands.AggregationFramework.AggregationResult` is a [dependent one](https://en.wikipedia.org/wiki/Dependent_type), used for the intermediary/MongoDB result, and must not be exposed as public return type in your application/API.

Then when calling `populatedStates(theZipCodeCol)`, the asynchronous result will be as bellow.

{% highlight javascript %}
[
  { "_id" -> "JP", "totalPop" -> 13185702 },
  { "_id" -> "NY", "totalPop" -> 19746227 }
]
{% endhighlight %}

> Note that for the state "JP", the population of Aogashima (200) and of Tokyo (13185502) have been summed.

As for the other commands in ReactiveMongo, it's possible to return the aggregation result as custom types (see [BSON readers](../bson/typeclasses.html)), rather than generic documents, for example considering a class `State` as bellow.

{% highlight scala %}
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

import reactivemongo.bson.Macros
import reactivemongo.api.collections.bson.BSONCollection

case class State(name: String, population: Long)

implicit val reader = Macros.reader[State]

def aggregate(col: BSONCollection): Future[col.BatchCommands.AggregationFramework.AggregationResult] = ???

def states(col: BSONCollection): Future[List[State]] =
  aggregate(col).map(_.result[State])
{% endhighlight %}

*Using cursor:*

The alternative [`aggregate1`](../api/index.html#reactivemongo.api.collections.GenericCollection@aggregate1[T]%28firstOperator:GenericCollection.this.PipelineOperator,otherOperators:List[GenericCollection.this.PipelineOperator],cursor:GenericCollection.this.BatchCommands.AggregationFramework.Cursor,explain:Boolean,allowDiskUse:Boolean,bypassDocumentValidation:Boolean,readConcern:Option[reactivemongo.api.ReadConcern],readPreference:reactivemongo.api.ReadPreference%29%28implicitec:scala.concurrent.ExecutionContext,implicitr:GenericCollection.this.pack.Reader[T]%29:scala.concurrent.Future[reactivemongo.api.Cursor[T]]) operation can be used, to process the aggregation result with a [`Cursor`](../api/index.html#reactivemongo.api.Cursor).

{% highlight scala %}
import scala.concurrent.{ ExecutionContext, Future }

import reactivemongo.bson._
import reactivemongo.api.Cursor
import reactivemongo.api.collections.bson.BSONCollection

def populatedStatesCursor(cities: BSONCollection)(implicit ec: ExecutionContext): Future[Cursor[BSONDocument]] = {
  import cities.BatchCommands.AggregationFramework
  import AggregationFramework.{ Cursor => AggCursor, Group, Match, SumField }

  val cursor = AggCursor(batchSize = 1) // initial batch size

  cities.aggregate1[BSONDocument](Group(BSONString("$state"))(
    "totalPop" -> SumField("population")), List(
    Match(document("totalPop" -> document("$gte" -> 10000000L)))),
    cursor)
}
{% endhighlight %}

**Average city population by state**

The Aggregation Framework can be used to find [the average population of the cities by state](http://docs.mongodb.org/manual/tutorial/aggregation-zip-code-data-set/#return-average-city-population-by-state).

In the MongoDB shell, it can be done as following.

{% highlight javascript %}
db.zipcodes.aggregate([
   { $group: { _id: { state: "$state", city: "$city" }, pop: { $sum: "$pop" } } },
   { $group: { _id: "$_id.state", avgCityPop: { $avg: "$pop" } } }
])
{% endhighlight %}

1. Group the documents by the combination of city and state, to get intermediate documents of the form `{ "_id" : { "state" : "NY", "city" : "NEW YORK" }, "pop" : 19746227 }`.
2. Group the intermediate documents by the `_id.state` field (i.e. the state field inside the `_id` document), and get the average of population of each group (`$avg: "$pop"`).

Using ReactiveMongo, it can be written as bellow.

{% highlight scala %}
import scala.concurrent.{ ExecutionContext, Future }

import reactivemongo.bson.{ BSONDocument, BSONString }
import reactivemongo.api.collections.bson.BSONCollection

def avgPopByState(col: BSONCollection)(implicit ec: ExecutionContext): Future[List[BSONDocument]] = {
  import col.BatchCommands.AggregationFramework.{
    AggregationResult, Avg, Group, Match, SumField
  }

  col.aggregate(Group(BSONDocument("state" -> "$state", "city" -> "$city"))(
    "pop" -> SumField("population")),
    List(Group(BSONString("$_id.state"))("avgCityPop" -> Avg("pop")))).
    map(_.documents)
}
{% endhighlight %}

**Largest and smallest cities by state**

Aggregating the documents can be used to find the [largest and the smallest cities for each state](http://docs.mongodb.org/manual/tutorial/aggregation-zip-code-data-set/#return-largest-and-smallest-cities-by-state):

{% highlight javascript %}
db.zipcodes.aggregate([
   { $group:
      {
        _id: { state: "$state", city: "$city" },
        pop: { $sum: "$pop" }
      }
   },
   { $sort: { pop: 1 } },
   { $group:
      {
        _id : "$_id.state",
        biggestCity:  { $last: "$_id.city" },
        biggestPop:   { $last: "$pop" },
        smallestCity: { $first: "$_id.city" },
        smallestPop:  { $first: "$pop" }
      }
   },

  // the following $project is optional, and
  // modifies the output format.

  { $project:
    { _id: 0,
      state: "$_id",
      biggestCity:  { name: "$biggestCity",  pop: "$biggestPop" },
      smallestCity: { name: "$smallestCity", pop: "$smallestPop" }
    }
  }
])
{% endhighlight %}

A ReactiveMongo function can be written as bellow.

{% highlight scala %}
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

import reactivemongo.bson.{ BSONDocument, BSONString, Macros }
import reactivemongo.api.collections.bson.BSONCollection

case class City(name: String, population: Long)
case class StateStats(state: String, biggestCity: City, smallestCity: City)

implicit val cityReader = Macros.reader[City]
implicit val statsReader = Macros.reader[StateStats]

def stateStats(col: BSONCollection): Future[List[StateStats]] = {
  import col.BatchCommands.AggregationFramework.{
    AggregationResult, Ascending, First, Group, Last, Project, Sort, SumField
  }

  col.aggregate(Group(BSONDocument("state" -> "$state", "city" -> "$city"))(
    "pop" -> SumField("population")),
    List(Sort(Ascending("population")), Group(BSONString("$_id.state"))(
        "biggestCity" -> Last("_id.city"), "biggestPop" -> Last("pop"),
        "smallestCity" -> First("_id.city"), "smallestPop" -> First("pop")),
      Project(BSONDocument("_id" -> 0, "state" -> "$_id",
        "biggestCity" -> BSONDocument("name" -> "$biggestCity",
          "population" -> "$biggestPop"),
        "smallestCity" -> BSONDocument("name" -> "$smallestCity",
          "population" -> "$smallestPop"))))).
  map(_.result[StateStats])
}
{% endhighlight %}

This function would return statistics like the following.

{% highlight scala %}
List(
  StateStats(state = "NY",
    biggestCity = City(name = "NEW YORK", population = 19746227L),
    smallestCity = City(name = "NEW YORK", population = 19746227L)),
  StateStats(state = "FR",
    biggestCity = City(name = "LE MANS", population = 148169L),
    smallestCity = City(name = "LE MANS", population = 148169L)),
  StateStats(state = "JP",
    biggestCity = City(name = "TOKYO", population = 13185502L),
    smallestCity = City(name = "AOGASHIMA", population = 200L)))
{% endhighlight %}

**Find documents using text indexing**

Consider the following [text indexes](https://docs.mongodb.org/manual/core/index-text/) is maintained for the fields `city` and `state` of the `zipcodes` collection.

{% highlight javascript %}
db.zipcodes.ensureIndex({ city: "text", state: "text" })
{% endhighlight %}

Then it's possible to find documents using the [`$text` operator](https://docs.mongodb.org/v3.0/reference/operator/query/text/#op._S_text), and also the results can be [sorted](https://docs.mongodb.org/v3.0/reference/operator/aggregation/sort/#metadata-sort) according the [text scores](https://docs.mongodb.org/v3.0/reference/operator/query/text/#text-operator-text-score).

For example to find the documents matching the text `"JP"`, and sort according the text score, the following query can be executed in the MongoDB shell.

{% highlight javascript %}
db.users.aggregate([
   { $match: { $text: { $search: "JP" } } },
   { $sort: { score: { $meta: "textScore" } } }
])
{% endhighlight %}

A ReactiveMongo function can be written as bellow.

{% highlight scala %}
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

import reactivemongo.bson.BSONDocument
import reactivemongo.api.collections.bson.BSONCollection

def textFind(coll: BSONCollection): Future[List[BSONDocument]] = {
  import coll.BatchCommands.AggregationFramework
  import AggregationFramework.{
    Cursor,
    Match,
    MetadataSort,
    Sort,
    TextScore
  }

  val firstOp = Match(BSONDocument(
    "$text" -> BSONDocument("$search" -> "JP")))

  val pipeline = List(Sort(MetadataSort("score", TextScore)))

  coll.aggregate1[BSONDocument](
    firstOp, pipeline, Cursor(1)).flatMap(_.collect[List]())
}
{% endhighlight %}

This will return the sorted documents for the cities `TOKYO` and `AOGASHIMA`.

**Random sample**

The [$sample](https://docs.mongodb.org/manual/reference/operator/aggregation/sample/) aggregation stage can be used (since MongoDB 3.2), in order to randomly selects documents.

In the MongoDB shell, it can be used as following to fetch a sample of 3 random documents.

{% highlight javascript %}
db.zipcodes.aggregate([
  { $sample: { size: 3 } }
])
{% endhighlight %}

With ReactiveMongo, the [Sample](../api/index.html#reactivemongo.api.commands.AggregationFramework@SampleextendsAggregationFramework.this.PipelineOperatorwithProductwithSerializable) stage can be used as follows.

{% highlight scala %}
import scala.concurrent.{ ExecutionContext, Future }
import reactivemongo.bson.BSONDocument
import reactivemongo.api.collections.bson.BSONCollection

def randomZipCodes(coll: BSONCollection)(implicit ec: ExecutionContext): Future[List[BSONDocument]] = {
  import coll.BatchCommands.AggregationFramework

  coll.aggregate(AggregationFramework.Sample(3)).map(_.head[BSONDocument])
}
{% endhighlight %}

*More:* The operators available to define an aggregation pipeline are documented in the [API reference](../../api/index.html#reactivemongo.api.commands.AggregationFramework).