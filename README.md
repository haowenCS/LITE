LITE is an auto-tuning system for various Spark applications on large-scale datasets. 


# Performance Overview
## Tuning Performance on Sparkbench Applications
We have conducted comparative experiments on 15 spark-bench applications. 

![*Figure 1 Percentage of execution time reduction by different tuning methods*](./fig1.png)
### Baselines ###
Figure 1 presents comparative performance of LITE, against several baselines. 
(1) Default: using Spark's default configurations. 

(2) Manual: manually tuning the applications based on expertise and the online tuning guides for maximally 12 hours. 

(3) BO: trial-and-error Bayesian Optimization tuning, where Gaussian Process was the surrogate model and Expected Improvement was the acquisition function. We used 5 most similar instances in the training set to initialize Gaussian Process. Then, BO was trained for at least 2 hours. 

(4) DDPG: a reinforcement learning framework based on Deep Deterministic Policy Gradient, where the action space was the configurations, and the state was the inner status summary of Spark. DDPG was also trained for at least 2 hours.

(5) DDPG+Code: similar as QTune, an additional module is incorporated in DDPG, which predicts the change of outer metric based on pretrained code features extracted by LITE and the inner status.

(6) RFR: a Random Forest Regression model is trained to map the input datasize and the application to an appropriate knob value. Since the prediction of RFR is numerical, for discrete valued knobs, we round the prediction of RFR to the nearest integer. 

### Evaluation Metrics ###
Figure 1 reports percentage of execution time reduction, which is defined as t/tmin, where t is the execution time produced by the method, and tmin is the smallest execution time by different methods on this application, according to Table 1. 

![*Table 1 Actural execution time by different methods*](./tab1_actural.png)


In summary, the average actural execution time and average percentage of execution time reduction for different tuning methods are:

| Method                                         | Default   | Manual    | DDPG(2h)  | DDPG+Code | BO(2h)    | RFR       | LITE     |
|------------------------------------------------|-----------|-----------|-----------|-----------|-----------|-----------|----------|
| Average Actural Execution Time                 | 7314.933  | 1329.733  | 2236.267  | 2244.933  | 1125.133  | 3055.067  |**588.594**|
| Average Percentage of Execution Time Reduction | 0.074     | 0.626     | 0.442     | 0.478     | 0.689     | 0.410     | **0.991**|


## Ablation Study on Feature Encoding and Performance Estimation Modules

![*Figure 2 Ranking performance by various feature encoding and performance estimation modules*](./fig2.png)

We used “feature encoding + performance estimation” to represent a comparative method. For example,𝑊 + 𝑆𝑉𝑅 means support vector regression was implemented on application instance features.

### Different Design Choices for Feature Encoding ###

(1) W: application instance features include the model’s input was application instance features (i.e., data features and environment features), output was the execution time
of the application instance.

(2) WC: in addition of the application instance features, we added program codes of the application as model’s input. 

(3) S: the model’s input was stagelevel features, while the model’s output was stage-level execution time. Besides the data features and environment features, we fed the
model with four key stage-level data statistics obtained in the Spark monitor UI, including stage input (i.e., bytes read from storage in this stage), stage output (i.e., bytes written in storage in this stage), shuffle read (i.e., total shuffle bytes and records read, includes both data read locally and data read from remote executors), and shuffle
write (i.e., bytes and records written to disk in order to be read by a shuffle in a future stage). Note that the stage-level data statistics were not used in our proposed model NECS, because they are only accessible when the application has been acturally executed on the real input data and it will be problematic to handle large-scale input data. 

(4) SC: in addition of the stage-level features, the stage-level codes were input in the manner of bag of words. 

(5) SCG: in addition of the stage-level code features, we used pretrained scheduler features (i.e., scheduler DAGs are trained in a LSTM to predict “next” DAG to obtain the DAG embedding vector) as model’s input.

(6) LSTM: a Long-Short-Term-Memory network was used to encode the stage-level codes. 

(7) Transformer: a transformer network based on multi-head self attention was used to encode the stage-level codes.

(8) GCN: a Graph Convolutional Neural network was used to encode the stage-level scheduler DAGs. 

### Different Design Choices for Performance Estimation ###

(1) SVR: support vector regression.

(2) LightGBM: an ensemble model based on gradient boosted machine. 

(3) XGB ranker: a ranking model.

(4) MLP: a tower Multi-Layer-Perception

### Evaluation Metrics ###

We used ranking evaluation metrics for a thorough study of the performance. Two evaluation metrics were adopted, HR@K and NDCG@K (here we set K=5).

![*Table 2 Ranking performance by different feature encoding and performance estimation modules*](./tab2.png)

In summary, the average HR@5 and NDCG@5 increasement over different clusters are

| Code Feature Encoding  | Workload Code | Stage Level Code (BOW) | Stage Level Code (LSTM) | Stage Level Code (Transformer) |
|------------------------|---------------|------------------------|-------------------------|--------------------------------|
| LITE's HR@5 Increase   | 0.065         | 0.046                  | 0.036                   | 0.042                          |
| LITE's NDCG@5 Increase | 0.076         | 0.044                  | 0.010                   | 0.027                          |

| DAG Encoding         | SCG (LSTM) |
|----------------------|------------|
| LITE's HR@5 Increase | 0.052      |
| LITE's HR@5 Increase | 0.073      |



# LITE 
## Enviroment

The project is implemented using python 3.6 and tested in Linux environment. Our system environment and cuda versions are as follows:

```
ubuntu 18.04
Hadoop 2.7.7
spark 2.4.7
HiBench 7.0
JAVA 1.8
SCALA  2.12.10
Python 3.6
Maven 4.15
CUDA Version: 10.1
```

## 1. Data Generate

To generate the training data for your own cluster environments, first install SparkBench https://github.com/CODAIT/spark-bench.

### 1.1. Run Applications

Run SparkBench applications using the following command:

```
python scripts/bo_sample.py <path_to_spark_bench_folders>
```

The log files will be saved on your spark history server, you can use the following command to download it:

```
hdfs dfs -get <path_to_spark_history_server_folders_on_hdfs> <local_folders>
```

### 1.2. Process Logs (for baselines)

Parse the log files generated by spark-bench to obtain stage status

```
python scripts/history_by_stage.py <spark_bench_conf_path> <history_dir> <result_path>
```

Note that the log file does not contain the data volume features of the workload, we need to add the data volume features through the configuration file in <spark_bench_conf_path>, result is like:

```json
{
	"AppId": "application_1616731661908_1527",
	"AppName": "SVM Classifier Example",
	"Duration": 23656,
	"SparkParameters": ["spark.default.parallelism=6", "spark.driver.cores=6"......],
	"StageInfo": {
		"0": {
			"duration": 3234,
			"input": 134283264,
			"output": 0,
			"read": 0,
			"write": 0
		}......
	},
	"WorkloadConf": ["NUM_OF_EXAMPLES=1600000", "NUM_OF_FEATURES=100"]
}
```

Save as training data file 

```
python scripts/build_dataset.py <result_path> <dataset_path>
```
The dataset files are written as comma-separated values files with a single header row. 

```
AppId	AppName	Duration	spark.default.parallelism	spark.driver.cores	spark.driver.memory	spark.driver.maxResultSize	spark.executor.instances	spark.executor.cores	spark.executor.memory	spark.executor.memoryOverhead	spark.files.maxPartitionBytes	spark.memory.fraction	spark.memory.storageFraction	spark.reducer.maxSizeInFlight	spark.shuffle.compress	spark.shuffle.file.buffer	spark.shuffle.spill.compress	rows	cols	itr	partitions	stage_id	duration	input	output	read	write	code	node_num	cpu_cores	cpu_freq	mem_size	mem_speed	net_width
```

### 1.3.Stage-based Code Organization (for LITE)

Obtain the stage code characteristics: enter the instrumentation folder, maven it into a jar package which name is preMain-1.0.jar 
```
cd instrumentation 
mvn clean package
```
and add  the package to the spark-submit's shell file as follow:
```sh
spark-submit --class <workload_class> --master yarn --conf "spark.executor.cores=4"  --conf "spark.executor.memory=5g" --conf "spark.driver.extraJavaOptions=-javaagent:<path_to_your_instrumentation_jar>/preMain-1.0.jar" <path_to_spark_bench>/<workload>/target/spark-example-1.0- SNAPSHOT.jar
```

The stage code is in /inst_log, you can change it by yourself; The file you get by instrumentation should be parsed by prediction_ml/spark_tuning/by_stage/instrumentation/all_code_by_stage/[get_all_code.py](https://github.com/cheyennelin/LITE/blob/main/prediction_ml/spark_tuning/by_stage/instrumentation/all_code_by_stage/get_all_code.py)

```
python get_all_code.py <folder_of_instrumentation> <code_reuslt_folder>
```

Our model is saved in [prediction_nn](https://github.com/cheyennelin/LITE/tree/main/prediction_nn),

You should use **data_process_text.py、dag2data.py、dataset_process.py** to get dictionary information、Process the edge and node information of the graph, and integrate all features.

```
python data_process_text.py <code_reuslt_folder>
python dag2data.py <log_folder>
python dataset_process.py
```

Dictionary information and graph information are saved in [dag_data](https://github.com/cheyennelin/LITE/tree/main/prediction_nn/dag_data), integrated them by [dataset_process.py](https://github.com/cheyennelin/LITE/blob/main/prediction_nn/dataset_process.py), the final dataset is in the folder dataset.

## 2.Model Training

Then use **fast_train.py** to train model.

```
python fast_train.py
```

You can change the config of model through [config.py](https://github.com/cheyennelin/LITE/blob/main/prediction_nn/config.py), and the model will be saved in the folder model_save.

## 3. Model Update

You can also use **trans_learn.py** to finetune the model.

```
python trans_learn.py
```

## 4.Model Testing

[nn_pred_1.py](https://github.com/cheyennelin/LITE/blob/main/prediction_nn/nn_pred_1.py) can test the model，we use [predict_first_cold.py](https://github.com/cheyennelin/LITE/blob/main/prediction_nn/predict_first_cold.py) to predict the best combination of parameters and evaluate the performance.

```
python nn_pred_8.py
python predict_first_cold.py
```
