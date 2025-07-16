#Real-Time Transaction Pattern Detection System

#1. Overview
This document outlines the architecture for a high-performance, scalable system designed to process a large stream of transaction data in near real-time. The system is composed of two primary, independent mechanisms, both running on Databricks:

•	Mechanism X: A scheduled Databricks job that reads a master transaction log file from an AWS S3 bucket, breaks it into smaller, manageable chunks, and writes them to a Delta table in a landing zone.

•	Mechanism Y: A continuous Databricks job that streams from the Delta table as new transaction chunks arrive, analyses them to detect specific, predefined patterns, and writes the results to a designated S3 bucket.


#2. System Architecture
The architecture is designed around a Databricks-centric model, using Delta Lake for reliable data transfer and external databases for stateful analysis.
 
<img width="1059" height="631" alt="image" src="https://github.com/user-attachments/assets/b329a340-dcf0-43ee-a65c-3f089c7512f2" />



#3. Mechanism X: 
Purpose
Mechanism X is responsible for breaking down a potentially massive, continuously growing source transaction file into small, consistently sized chunks using the power of pandas on Databricks. 
Process Flow
1.	Trigger: A Databricks Job, containing a notebook or script, is scheduled to run every 5 seconds.
2.	State Management: The job uses a simple mechanism (a small state file in S3) to remember the last line number (offset) it processed from the source - offset.csv.
3.	Chunk Creation: The job reads the source file, skips to the last known offset, and reads the next 10,000 transaction entries into a DataFrame.
4.	Output: The DataFrame is added as a chunk csv file located in the S3 Landing Zone (s3://your-bucket/landing_zone/). 
5.	State Update: The job updates its state file/table with the new last processed line number.

#4. Mechanism Y: Pattern Detector (Databricks Job)
Purpose
Mechanism Y is the core analytical engine. It runs as a continuous Databricks job, using Structured Streaming to process data as soon as it's available in the landing zone, perform complex pattern matching, and output actionable insights.
Process Flow
1.	Trigger: A continuous Databricks job is initiated, configured to use the landing zone  chunks as a streaming source. It automatically detects and processes new data added by Mechanism X.
2.	Ingestion: The streaming job reads new transaction chunks from the S3 folder in micro-batches.
3.	Stateful Analysis: For each transaction in the chunk, the job reads from and writes to into a delta table. This table is crucial for maintaining the state required to evaluate the patterns across multiple chunks. The table would store aggregates like:
o	Total transaction counts per merchant.
o	Total transaction counts for each customer with a specific merchant.
o	Running average transaction values for customers.
o	Gender-based customer counts for each merchant.
o	Data for calculating percentiles.
4.	Pattern Detection: The function evaluates the incoming transactions against the three patterns defined below.
5.	Buffering: Detections are collected within the Spark job. A foreachBatch sink is used to process the micro-batch of detections.
6.	Output: Within the foreachBatch operation, once 50 detections are collected, the job writes all 50 records to a single, unique CSV file in the detections S3 bucket (s3://your-bucket/detections/). The file is named uniquely, e.g., detections_1657886410.csv.

#5. Pattern Definitions

PatId1: UPGRADE
•	ActionType: UPGRADE
•	Conditions:
1.	The merchant's total number of transactions must exceed 50,000.
2.	The customer's total transaction count for that merchant must be in the top 10th percentile compared to all other customers of that same merchant.
3.	The customer's average transaction weight (averaged over all their transactions with that merchant) must be in the bottom 10th percentile.

PatId2: CHILD
•	ActionType: CHILD
•	Conditions:
1.	The customer has made at least 80 transactions with the specific merchant.
2.	The customer's average transaction value for that merchant is less than ₹23.
   
PatId3: DEI-NEEDED
•	ActionType: DEI-NEEDED
•	Conditions:
1.	The total number of female customers for the merchant is greater than 100.
2.	The total number of female customers is less than the total number of male customers for that same merchant.
•	Note: For this detection, customerName can be “” as the detection applies to the merchant, not an individual customer.


#6. Implementation Considerations
•	State Management: Delta table is used to gather the coming data from chunks and using Py spark on it to analyse for the detections in the Databricks job.
•	Job Management: Both Databricks jobs need to be monitored for failures. Configure job failure notifications and retry policies appropriately.
•	Deployment: The entire infrastructure, including Databricks jobs and cloud resources, should be managed as code using a framework like Terraform for repeatable and reliable deployments.

