---
title: Hyperparameter Tuning
---

An engine often depends on a number of parameters, for example, naive bayesian
classification algorithm has a smoothing parameter to make the model more
adaptive to unseen data. Comparing with parameters which are *learnt* by the
machine learning algorithm, this smoothing parameter *teaches* how the algorithm
works. Therefore, they are usually called *hyperparameters*.

In PredictionIO, we always take a holistic view of an engine. An engine is
comprised of a set of ***DAS*** controllers, as well as the parameter for each
controller.
In the evaluation, we attempt to find out the best hyperparameters for an
*engine*, which we call ***engine params***. With an engine params, we can
deploy a complete engine.

This section demonstrates how to select the optimal engine params
whilst ensuring the model doesn't overfit using PredictionIO's evaluation
module.

## The Evaluation Design

The PredictionIO evaluation module finds out the best engine params for an
engine.

Given a engine params, we instantiate an engine and evaluate it with existing data.
The data is splitted into two sets, a training set and a validation set. 
The training
set is used to train the engine, the same way as how we deploy a prediction
engine described in earlier sections. The validation set is used to validate the
engine, we query the engine with the test set data. 
We define a ***metric*** to compare ***predicted result*** returned from
the engine with the ***actual result*** which we obtained from the test data.
The goal is to maximize the metric score.

This process is repeated many times with a series of engine params.
At the end, PredictionIO returns the best engine params.

We demonstrate the evaluation with [the classification template]
(/templates/classification/quickstart/).

## Evaluation Data Generation

In evaluation data generation, the goal is to generate a sequence of (training, 
validation) data tuple. A common way is to use a *k-fold* generation process. 
The data set is splitted into *k folds*. We generate k tuples of training and
validation sets, for each tuple, the training set takes *k - 1* of the folds and
the validation set takes the remaining fold.

To enable evaluation data generation, we need to define the ***actual result***
and implement the method for generating the (traingin, validation) data tuple.

### Actual Result

In MyClassification/src/main/scala/***Engine.scala***, the `ActualResult` class
defines the ***actual result***:

```scala
class ActualResult(
  val label: Double
) extends Serializable
```

This class is used to store the actual label of the data (contrast to
`PredictedResult` which is output of the engine).

### Implement Data Generation Method in DataSource

In MyClassification/src/main/scala/***DataSource.scala***, the method
`readEval` method reads, and selects, data from datastore and returns a 
sequence of (training, validation) data.

```scala
class DataSource(val dsp: DataSourceParams)
  extends PDataSource[TrainingData, EmptyEvaluationInfo, Query, ActualResult] {
  
  ...

  override
  def readEval(sc: SparkContext)
  : Seq[(TrainingData, EmptyEvaluationInfo, RDD[(Query, ActualResult)])] = {
    require(!dsp.evalK.isEmpty, "DataSourceParams.evalK must not be None")

    // The following code reads the data from data store. It is equivalent to
    // the readTraining method. We copy-and-paste the exact code here for
    // illustration purpose, a recommended approach is to factor out this logic
    // into a helper function and have both readTraining and readEval call the
    // helper.
    val eventsDb = Storage.getPEvents()
    val labeledPoints: RDD[LabeledPoint] = eventsDb.aggregateProperties(
      appId = dsp.appId,
      entityType = "user",
      // only keep entities with these required properties defined
      required = Some(List("plan", "attr0", "attr1", "attr2")))(sc)
      // aggregateProperties() returns RDD pair of
      // entity ID and its aggregated properties
      .map { case (entityId, properties) =>
        try {
          LabeledPoint(properties.get[Double]("plan"),
            Vectors.dense(Array(
              properties.get[Double]("attr0"),
              properties.get[Double]("attr1"),
              properties.get[Double]("attr2")
            ))
          )
        } catch {
          case e: Exception => {
            logger.error(s"Failed to get properties ${properties} of" +
              s" ${entityId}. Exception: ${e}.")
            throw e
          }
        }
      }.cache()
    // End of reading from data store

    // K-fold splitting
    val evalK = dsp.evalK.get
    val indexedPoints: RDD[(LabeledPoint, Long)] = labeledPoints.zipWithIndex

    (0 until evalK).map { idx => 
      val trainingPoints = indexedPoints.filter(_._2 % evalK != idx).map(_._1)
      val testingPoints = indexedPoints.filter(_._2 % evalK == idx).map(_._1)

      (
        new TrainingData(trainingPoints),
        new EmptyEvaluationInfo(),
        testingPoints.map { 
          p => (new Query(p.features.toArray), new ActualResult(p.label)) 
        }
      )
    }
  }
}
```

The `readEval` method returns a sequence of (`TrainingData`, `EvaluationInfo`,
`RDD[(Query, ActualResult)]`. 
`TrainingData` is the same class we use for deploy,
`RDD[(Query, ActualResult)]` is the 
validation set, `EvaluationInfo` can be used to hold some global evaluation data
, it is not used in the current example.

Lines 11 to 41 is the logic of reading and transformating data from the
datastore, it is equvialent to the existing `readTraining` method. After line
41, the variable `labeledPoints` contains the complete dataset with which we use
to generate the (training, validation) sequence.

Lines 43 to 57 is the *k-fold* logic. Line 45 gives each data point a unique id,
and we decide whether the point belongs to the training or validation set
depends on the *mod* of the id (lines 48 to 49).
For each point in the validation set, we construct the `Query` and
`ActualResult` (line 55) which is used validate the engine.

## Evaluation Metrics

We define a `Metric` which gives a *score* to engine params, the higher the
score, the better the engine params is. In the classification template, we use
the [*Precision* score](http://en.wikipedia.org/wiki/Precision_and_recall),
which measure the portion of correct prediction among all data points.

In MyClassification/src/main/scala/**Evaluation.scala**, the class
`Precision` implements the *precision* score. 
It extends a base helper class `AverageMetric` which calculates the average
score overall *(Query, PredictionResult, ActualResult)* tuple.

```scala
case class Precision 
  extends AverageMetric[
      EmptyEvaluationInfo, Query, PredictedResult, ActualResult] {
  def calculate(query: Query, predicted: PredictedResult, actual: ActualResult)
  : Double = (if (predicted.label == actual.label) 1.0 else 0.0)
}
```

Then, implement a `Evaluation` object to define the engine and metric 
used in this evaluation.

```scala
object PrecisionEvaluation extends Evaluation {
  engineMetric = (ClassificationEngine(), new Precision())
}
```

## Parameters Generation
The last component is to specify the list of engine params we want to evaluate.
In this guide, we discuss the simplest method. We specify an explicit list of
engine params to be evaluated. 

In MyClassification/src/main/scala/**Evaluation.scala**, the object
`EngineParamsList` specifies the engine params list to be used.

```scala
object EngineParamsList extends EngineParamsGenerator {
  // Define list of EngineParams used in Evaluation

  // First, we define the base engine params. It specifies the appId from which
  // the data is read, and a evalK parameter is used to define the
  // cross-validation.
  private[this] val baseEP = EngineParams(
    dataSourceParams = DataSourceParams(appId = 18, evalK = Some(5)))

  // Second, we specify the engine params list by explicitly listing all
  // algorithm parameters. In this case, we evaluate 3 engine params, each with
  // a different algorithm params value.
  engineParamsList = Seq(
    baseEP.copy(algorithmParamsList = Seq(("naive", AlgorithmParams(10.0)))),
    baseEP.copy(algorithmParamsList = Seq(("naive", AlgorithmParams(100.0)))),
    baseEP.copy(algorithmParamsList = Seq(("naive", AlgorithmParams(1000.0)))))
}
```

A good practise is to first define a base engine params, it contains the common
parameters used in all evaluations (lines 7 to 8). With the base params, we 
construct the list of engine params we want to evaluation by 
adding or replacing the controller parameter. Lines 13 to 16 generate 3 engine
parameters, each has a different smoothing parameters.

## Running the Evaluation

It remains to run the evaluation. The command `pio eval` is used. It takes to
mandatory parameter, 1. the `Evaluation` object, and 2. the
`EngineParamsGenerator`. The following command kickstarts the evaluation 
workflow for the classification template.

### Building
```
$ pio build
```

### Running the evaluation

```
$ pio eval org.template.classification.PrecisionEvaluation \
    org.template.classification.EngineParamsList 
```

You will see the following output:

```
...
[INFO] [CoreWorkflow$] CoreWorkflow.runEvaluation
...
[INFO] [EvaluationWorkflow$] Iteration 0
[INFO] [EvaluationWorkflow$] EngineParams: {"dataSourceParams":{"":{"appId":18,"evalK":5}},"preparatorParams":{"":{}},"algorithmParamsList":[{"naive":{"lambda":10.0}}],"servingParams":{"":{}}}
[INFO] [EvaluationWorkflow$] Result: 0.9281045751633987
[INFO] [EvaluationWorkflow$] Iteration 1
[INFO] [EvaluationWorkflow$] EngineParams: {"dataSourceParams":{"":{"appId":18,"evalK":5}},"preparatorParams":{"":{}},"algorithmParamsList":[{"naive":{"lambda":100.0}}],"servingParams":{"":{}}}
[INFO] [EvaluationWorkflow$] Result: 0.9150326797385621
[INFO] [EvaluationWorkflow$] Iteration 2
[INFO] [EvaluationWorkflow$] EngineParams: {"dataSourceParams":{"":{"appId":18,"evalK":5}},"preparatorParams":{"":{}},"algorithmParamsList":[{"naive":{"lambda":1000.0}}],"servingParams":{"":{}}}
[INFO] [EvaluationWorkflow$] Result: 0.4444444444444444
...
[INFO] [CoreWorkflow$] Stop spark context
[INFO] [CoreWorkflow$] Optimal score: 0.9281045751633987
[INFO] [CoreWorkflow$] Optimal engine params: {
  "dataSourceParams":{
    "":{
      "appId":18,
      "evalK":5
    }
  },
  "preparatorParams":{
    "":{
      
    }
  },
  "algorithmParamsList":[
    {
      "naive":{
        "lambda":10.0
      }
    }
  ],
  "servingParams":{
    "":{
      
    }
  }
}
...
```

The console prints out the evaluation metric score of each engine params, and
finally pretty print the optimal engine params.
Amongst the 3 engine params we evaluate, *lambda = 10.0* yields the highest 
precision score of 0.9281. 


## Notes

- We deliberately not metion ***test set*** in this hyperparameter tuning guide.
In machine learning literature, the ***test set*** is a separate piece of data
which is used to evaluate the final engine params outputted by the evaluation
process. This guarantees that no information in the training / validation set is
*leaked* into the engine params and yields a biased outcome. With PredictionIO,
there are multiple ways of conducting robust tuning, we will cover this
topic in the coming sections.
