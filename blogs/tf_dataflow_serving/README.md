# Serving TensorFlow Models for Dataflow 


## Repo Structure

1. **[model](https://github.com/GoogleCloudPlatform/training-data-analyst/tree/master/blogs/tf_dataflow_serving/model)**: The directory inclueds:
    1. Trained and exported TensorFlow Model (babyweight estimator)
    2. **inference.py** module, which includes two "estimate" functions
        1. using a model API deployed on **Cloud ML Engine**
        2. using the **local exported TF model**.
2. **[pipelines](https://github.com/GoogleCloudPlatform/training-data-analyst/tree/master/blogs/tf_dataflow_serving/piplines)**: The directory includes:
    1. **batch_process.py** module, which runs apache beam batch pipeline (source is BigQuery and sink is Cloud Storage):
        1. one performs prediction during the processing (either uding Cloud ML Engine API or local model).
        2. the other prepares the data from BigQuery to Cloud Storage for a subsequent Cloud ML Engine "Batch Predict" job.
    2. **stream_process.py** module, which runs apache beam stream pripeline (source is PubSub and sink is BigQuery):
        1. one performs "data point by data point" prediction (either uding Cloud ML Engine API or local model).
        2. the other creates "micro-batches" of the data points before sent for prediction (given a window size).
3. **[tests](https://github.com/GoogleCloudPlatform/training-data-analyst/tree/master/blogs/tf_dataflow_serving/tests)**: The directory includes:
    1. **inference_test.py** module to test the model/inference.py functionality.
    2. **pubsub_pull_test.py** module to test that messages are sent and received to the pubsub topic.
4. **[scripts](https://github.com/GoogleCloudPlatform/training-data-analyst/tree/master/blogs/tf_dataflow_serving/scriptys)**: The directory includes:
    1. **deploy-cmle.sh** command-line script to deploy the exported TF model to Cloud ML Engine
    2. **predict-batch-cmle.sh** command-line script to submit a "Batch Predict" job to Cloud ML Engine (to be executed by **run_pipeline_with_batch_predict** function in **batch_process.py** module).
    3. **bq_stats.sql** SQL script to query the stats of the streamed data into BigQuery
5. **Root Files**
    1. **run_pipeline.py** executes a pipeline based on given parameters:
        1. EXPERIMENT: <"batch" | "batch-predict" | "stream" | "stream-m-batches" >
        2. INFERENCE_TYPE: <"local" | "cmle" | "NA">
    2. **simulate_stream.py** sends data points to a specified pubsub topic
    3. **setup.py**
    4. **requirements.txt**
    5. **README.md**
 

## Approaches

### Batch (Source: BigQuery -> Sink: Cloud Storage)
1. Calling a local TF Model with-in Dataflow Batch Pipeline
2. Calling CMLE for each data point in Dataflow Batch Pipeline
3. Dataflow to prepare data from BigQuery to GCS, then submit Batch Prediction Job to CMLE
### Stream (Source Pubs/Sub -> Sink: BigQuery)
1. Calling a local TF Model with-in Dataflow Streaming Pipeline
2. Calling CMLE for each data point in Dataflow Streaming Pipeline
### Stream + Micro-batching (Source Pubs/Sub -> Sink: BigQuery)
1. Batching data points then send them to CMLE in Dataflow Streaming Pipeline
2. Batching data points then send them to TF Model with-in in Dataflow Streaming Pipeline


## Results

 . Max Workers in Dataflow = 20 
 . Max Workers in Cloud ML Engine = 20

| Batch |                 | DF then Batch CMLE | DF + Online CMLE Prediction | TF Model in DF |
|---------------------|-----------------|--------------------|-----------------------------|----------------|
| Dataset = 10k Rows  | Running Time    |  (DF) + (MLE) =                  |      19 min 21 sec            |                |
|                     | Total vCPU Time |  (DF) + (MLE) =                 |      3.633 hr + CMLE           |                |
|                     | Cost            |  (DF) + (MLE) =                  |                               |                |
| Dataset = 100k Rows | Running Time    |  (DF) + (MLE) =                  |  Quota Limits after 1 hr      |                |
|                     | Total vCPU Time |  (DF) + (MLE) =                 |       8.119 hr + CMLE          |                |
|                     | Cost            |  (DF) + (MLE) =                  |                               |                |
| Dataset = 1M Rows   | Running Time    |  (DF) + (MLE) =                  |        NA                     |                |
|                     | Total vCPU Time |  (DF) + (MLE) =                 |        NA                      |                |
|                     | Cost            |  (DF) + (MLE) =                  |        NA                     |                |
| Dataset = 10M Rows  | Running Time    |  (DF) + (MLE) =                  |        NA                     |                |
|                     | Total vCPU Time |  (DF) + (MLE) =                  |        NA                     |                |
|                     | Cost            |  (DF) + (MLE) =                  |        NA                     |                |


Note that, if the sink is, for example, BQ or Cloud SQL, the "DF then Batch CMLE" approach would need another step 
to read data from GCS, transform (if needed) and load the data to such a sink, 
while "TF Model in DF" would just need to change the sink.

| Stream    (1K Messages)                 |              | DF + Online CMLE | TF Model in DF |
|-----------------------------------------|--------------|------------------|----------------|
|  10 Messages Per Sec      (1 node)      | Running Time |                  |                |
|                                         | Min. Latency |                  |                |
|                                         | Max. Latency |                  |                |
|                                         | Avg. Latency |                  |                |
|                                         | Cost         |                  |                |
|  100 Messages Per Sec    (5 nodes)      | Running Time |                  |                |
|                                         | Min. Latency |                  |                |
|                                         | Max. Latency |                  |                |
|                                         | Avg. Latency |                  |                |
|                                         | Cost         |                  |                |
|  1000 Messages Per Sec   (10 nodes)     | Running Time |                  |                |
|                                         | Min. Latency |                  |                |
|                                         | Max. Latency |                  |                |
|                                         | Avg. Latency |                  |                |
|                                         | Cost         |                  |                |



| Stream + Micro-batches           | Dataset = 5K Rows, Frequency = 10 Messages Per Sec | DF + Online CMLE | TF Model in DF |
|------------------|--------------------------------------|------------------|----------------|
| Batch Size = 10  | Running Time                         |                  |                |
|                  | Min. Latency                         |                  |                |
|                  | Max. Latency                         |                  |                |
|                  | Avg. Latency                         |                  |                |
|                  | Cost                                 |                  |                |
| Batch Size = 150 | Running Time                         |                  |                |
|                  | Min. Latency                         |                  |                |
|                  | Max Latency                          |                  |                |
|                  | Avg. Latency                         |                  |                |
|                  | Cost                                 |                  |                |
| Batch Size = 500 | Running Time                         |                  |                |
|                  | Min. Latency                         |                  |                |
|                  | Max. Latency                         |                  |                |
|                  | Avg. Latency                         |                  |                |
|                  | Cost                                 |                  |                |