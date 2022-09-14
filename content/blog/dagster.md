---
title: "Building ML pipelines using Dagster"
date: 2022-09-09
draft: false
description: "Using workflow orchestration to streamline pipeline development"
comments: false
tags: ['machine learning']
---

Suppose we are in the business of building machine learning pipelines. We extract insights from raw data using ML models and present these in a format that is valuable to an end user. There are many parts of this pipeline that could change over time, and we would like to handle these changes effectively. What changes are we talking about here? I see 3 main parts.

**1. Input data (aka test data)**: How does the volume of input data vary over time? Is it received as a stream or in batches? [Does the distribution of this data remain stable?]({{< relref "data-drift">}})

**2. Models**: As more accurate models are developed, we will want to switch out the specific models being used at each stage. How can we make the pipeline model-agnostic? How do we load models efficiently?

**3. Output data**: As well as results, we want to output metrics, errors, metadata, etc. Where will these be stored? How will they be formatted?


## Solution

Enter Dagster. Dagster is a workflow orchestration tool which makes it easier to build, test, and debug pipelines. It also has features to schedule and track runs. Here is an overview of how it works.

An ML pipeline can be thought of as a set of operations. Some must happen sequentially and others can happen in parallel. These operations can be represented in a directed acyclic graph (![DAG](https://medium.com/hashmapinc/building-ml-pipelines-8e27344a42d2)).

Dagster has a number of classes which help implement ML pipelines as DAGs.
- Op: A function representing a node in the DAG
- Graph: A function representing the connections between nodes
- Resource: A function that can be called by any node

![Dagit UI showing DAG](/img/dagster/dagit-dag.png "Dagit UI showing DAG of ops[1]")

### 1. Changing input data

To get the right input data, dagster implements schedules and sensors. Schedules trigger pipeline runs at specific intervals. For example, we may want to process data about all trades made on a financial instrument at the end of each working day. Sensors trigger runs based on changes in external state. For example, we may want to process data every time a user registers on our website. Sensors also have a cursor property to cache data about the last piece of processed data. This allows them to trigger runs based on any **new** data in a persistent store by only processing data uploaded after the time stored in the cursor.

By extending Dagster types, we can do data validation in the channels between ops. For example, if we are passing data around in Pandas dataframes, we can make use of the [dagster-pandas](https://docs.dagster.io/integrations/pandas) library. Dagster has many such [integrations](https://docs.dagster.io/integrations).

### 2. Changing models
Resources are a great tool to deal with changing models. To test our pipeline in different environments, we can create resources that interface with different data stores and load models using the schema enforced by that data store. In testing, we may load up models stored as pickle files from the local filesystem but in deployment we may use a different resource to load models from weights stored in Amazon S3. These resources can then be used by any op in the DAG.

### 3. Changing output data
For some background context, dagster caches artifacts in between ops which makes it easy to rerun a failed job from the point of failure. This speeds up development massively. The artifact store is managed by the IO manager and IO managers can be overridden. By writing new IO managers to interface with other data stores or registries like MLFlow, we can read input data and models as well as write outputs, metrics, and metadata to the most appropriate location.

During operations, Dagster can emit various events. Failure events cancel the current job if specified conditions are met and allow error messages to be customised. Asset observations keep track of metadata as assets move through the pipeline. The Dagit UI provides great visualisations for both of these, but I've found accessing event store data directly to be clunky in the version of Dagster I've been using (1.0.8).

![Dagit UI showing a run](/img/dagster/dagit-run.png "Dagit UI showing a run[2]")

## Conclusion

All in all, dagster has a host of great features that enable certain key outcomes:
- Faster development by enabling pipelines to be rerun from arbitrary points and by making pipelines environment-agnostic. The environment is dictated by the _resources_ used.
- Automated run triggering using _schedules_ and _sensors_
- Faster debugging through _events_. Specifically, _observation_ and _failure_ events.

### Extra homework

There are some features that I've not mentioned here but definitely recommending reading about.
- Advanced DAG options like conditional branching and dynamic outputs
- Hooks and retries to define success and failure callback policies on ops
- The Dagit UI. It nicely visualises your DAGs, runs, configurations, assets, sensors, etc. It also makes it easier to sift through previous runs

## References

[1] https://docs.dagster.io/concepts/dagit/dagit

[2] https://blog.devgenius.io/build-etl-pipelines-with-dagster-4c5f2ac678db