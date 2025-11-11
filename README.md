# ITCS-6190 Assignment 3: AWS Data Processing Pipeline

**AWS Core Services**

This repository demonstrates a concise, reproducible serverless data-processing pipeline on AWS: ingest CSV files into S3, automatically clean and filter them using Lambda, catalog the processed data with Glue so it can be queried using Athena, and present query results on a simple EC2-hosted web dashboard. Below each step, you will find a brief explanation of *what* the step accomplishes, a precise *approach* to complete it, and related screenshots.

Prerequisites
- An AWS account with permissions to create S3, Lambda, IAM roles, Glue, Athena, and EC2 resources.
- The Orders.csv input file (placed in the `raw/` S3 prefix).
- Basic familiarity with the AWS Console and SSH.

## 1. Amazon S3 Bucket Structure 🪣

S3 is used as the pipeline's durable storage. We separate **raw, processed, and enriched** outputs so each component reads from/ writes to a single, well-defined location.
  - This separation enforces clear ownership for each stage (ingest, processing, analytics) and makes it easy to apply different lifecycle or access policies per stage.
  - It simplifies debugging because you can inspect the raw input independently of transformed outputs and replay or reprocess raw files if needed.
  - Using prefixes rather than separate buckets reduces cross-bucket permissions complexity while still allowing prefix-scoped policies.

**Approach:** Create an S3 bucket with the following folder structure to manage the data workflow:

* **`bucket-name/`**
    * **`raw/`**: For incoming raw data files.
    * **`processed/`**: For cleaned and filtered data output by the Lambda function.
    * **`enriched/`**: For storing athena query results.

<img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/2158d2a7-9ae2-4b92-a54e-d734aa36b74b" />


---

## 2. IAM Roles and Permissions 🔐

IAM roles ensure that each AWS service has the correct permissions to interact securely with others. 
-The Lambda Execution Role allows Lambda to read from and write to S3. 
-The Glue Service Role allows the crawler to access data and create metadata. 
-The EC2 Instance Profile enables the hosted web server to query Athena and read output files from S3.

**Approach:** Create the following IAM roles to grant AWS services the necessary permissions to interact with each other securely.

### Lambda Execution Role

1.  Navigate to **IAM** -> **Roles** and click **Create role**.
2.  **Trusted entity type**: Select **AWS service**.
3.  **Use case**: Select **Lambda**.
4.  **Add Permissions**: Attach the following managed policies:
    * `AWSLambdaBasicExecutionRole`
    * `AmazonS3FullAccess`
5.  Give the role a descriptive name (e.g., `Lambda-S3-Processing-Role`) and create it.

<img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/dda84c2f-cc2e-41b4-b925-0f75d0d5e588" />


### Glue Service Role

1.  Create another IAM role for **AWS service** with the use case **Glue**.
2.  **Add Permissions**: Attach the following policies:
    * `AmazonS3FullAccess`
    * `AWSGlueConsoleFullAccess`
    * `AWSGlueServiceRole`
3.  Name the role (e.g., `Glue-S3-Crawler-Role`) and create it.

 <img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/216893d0-8dca-49d9-9b0f-95f56ba04b7e" />


### EC2 Instance Profile

1.  Create a final IAM role for **AWS service** with the use case **EC2**.
2.  **Add Permissions**: Attach the following policies:
    * `AmazonS3FullAccess`
    * `AmazonAthenaFullAccess`
3.  Name the role (e.g., `EC2-Athena-Dashboard-Role`) and create it.

<img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/10deb663-feb7-4ccb-acbe-c6ed708d713e" />

---

## 3. Create the Lambda Function ⚙️

The Lambda function is responsible for processing data as soon as it is uploaded. It reads raw input files from the **raw/** folder of the S3 bucket ```cloudassignmentsudeepta```, applies filtering or transformation, and writes the cleaned dataset to the **processed/** folder of the same S3 bucket. This automatic execution ensures efficient and hands-free data preparation.

**Approach:** This function will automatically process files uploaded to the `raw/` S3 folder.

1.  Navigate to the **Lambda** service in the AWS Console.
2.  Click **Create function**.
3.  Select **Author from scratch**.
4.  **Function name**: `FilterAndProcessOrders`
5.  **Runtime**: Select **Python 3.9** (or a newer version).
6.  **Permissions**: Expand *Change default execution role*, select **Use an existing role**, and choose the **Lambda Execution Role** you created.
7.  Click **Create function**.

 <img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/184f0f21-c867-456d-a1ce-624fee240362" />
 

   
8.  In the **Code source** editor, replace the default code with the LambdaFunction.py code for processing the raw data.
   


<img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/26b7e738-7b06-443a-990c-7fe46e239efc" />


---

## 4. Configure the S3 Trigger ⚡

The S3 trigger ensures the Lambda function runs automatically whenever a new file is uploaded. By restricting the trigger to the **raw/** folder and **.csv** files, the system ensures that only relevant data initiates processing. This removes the need for manual execution or scheduled runs.

**Approach:** Set up the S3 trigger to invoke your Lambda function automatically.

1.  In the Lambda function overview, click **+ Add trigger**.
2.  **Source**: Choose **S3**.
3.  **Bucket**: Select your S3 bucket.
4.  **Event types**: Choose **All object create events**.
5.  **Prefix (Required)**: Enter `raw/`. This ensures the function only triggers for files in this folder.
6.  **Suffix (Recommended)**: Enter `.csv`.
7.  Check the acknowledgment box and click **Add**.

<img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/83581225-2ba2-450b-87e8-d5244c7d55b8" />

--- 
**Start Processing of Raw Data**: Now upload the Orders.csv file into the `raw/` folder of the S3 Bucket. This will automatically trigger the Lambda function.

<img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/4c9586fe-4220-4b07-b332-812674c51325" />


---

**Generation of Processed Data**: Once the raw file is uploaded and the Lambda function is triggered, the raw data records will then be processed using the logic mentioned in the Lambda function, and a new processed file will be generated in the `processed/` folder of the S3 Bucket.



<img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/ce7c6e49-be5a-4298-a886-e944c2b6f964" />


---

## 5. Create a Glue Crawler 🕸️

The Glue Crawler scans the processed data and generates a schema-based table in the Glue Data Catalog. This table allows Athena to recognize and query the cleaned dataset. The crawler ensures that any new data added to the **processed/** folder becomes automatically queryable.

**Approach:** The crawler will scan your processed data and create a data catalog, making it queryable by Athena.

1.  Navigate to the **AWS Glue** service.
2.  In the left pane, select **Crawlers** and click **Create crawler**.
3.  **Name**: `orders_processed_crawler`.
4.  **Data source**: Point the crawler to the `processed/` folder in your S3 bucket.
5.  **IAM Role**: Select the **Glue Service Role** you created earlier.
6.  **Output**: Click **Add database** and create a new database named `orders_db`.
7.  Finish the setup and run the crawler. It will create a new table in your `orders_db` database.

<img width="1470" height="846" alt="image" src="https://github.com/user-attachments/assets/c8b7a83c-cf9a-404f-9d75-f5aaad3ba317" />


---

## 6. Query Data with Amazon Athena 🔍

Athena enables SQL-based querying directly on data stored in S3, using the metadata from Glue. You can perform analytical operations like aggregations, summaries, and ranking to generate insights. These queries form the core metrics that will be displayed on the final web dashboard.

**Approach:** Navigate to the **Athena** service. Ensure your data source is set to `AwsDataCatalog` and the database is `orders_db`. You can now run SQL queries on your processed data.

**Queries to be executed:**
* **Total Sales by Customer**: Calculate the total amount spent by each customer.

<img width="1920" height="1002" alt="image" src="https://github.com/user-attachments/assets/bc76a3d6-6f12-4b94-a04b-d2b169b3b48f" />

  
* **Monthly Order Volume and Revenue**: Aggregate the number of orders and total revenue per month.

<img width="1920" height="1002" alt="image" src="https://github.com/user-attachments/assets/5e822e44-95bd-4c85-ba34-08c50024c137" />
  
* **Order Status Dashboard**: Summarize orders based on their status (`shipped` vs. `confirmed`).

<img width="1920" height="1002" alt="image" src="https://github.com/user-attachments/assets/a4f8919b-6933-459d-afbb-943b5cf667d8" />

* **Average Order Value (AOV) per Customer**: Find the average amount spent per order for each customer.

<img width="1920" height="1002" alt="image" src="https://github.com/user-attachments/assets/88ba6bc2-2b91-492b-8940-c1a80da7b1fb" />
  
* **Top 10 Largest Orders in February 2025**: Retrieve the highest-value orders from a specific month.

<img width="1920" height="1002" alt="image" src="https://github.com/user-attachments/assets/c22e933a-13c5-4ce7-9e06-d02b035d6c87" />

**The results of all these 5 queries are stored in the ```Enriched/``` folder of the S3 bucket**

<img width="1920" height="1002" alt="image" src="https://github.com/user-attachments/assets/fa8e7ce8-0563-435c-9b13-c2818979c12f" />


---

## 7. Launch the EC2 Web Server 🖥️

The EC2 instance hosts a Flask-based web application that retrieves processed Athena results and displays them as a dashboard. Proper security and access configuration ensure the server is reachable externally while remaining secure. The IAM Instance Profile grants access to Athena and S3 data.

**Approach:** This instance will host a simple web page to display the Athena query results.

1.  Navigate to the **EC2** service and click **Launch instance**.
2.  **Name**: `Athena-Dashboard-Server`.
3.  **Application and OS Images**: Select **Amazon Linux 2023 AMI**.
4.  **Instance type**: Choose **t3.micro** (Free tier eligible).
5.  **Key pair (login)**: Create and download a new key pair. **Save the `.pem` file!**
6.  **Network settings**: Click **Edit** and configure the security group:
    * **Rule 1 (SSH)**: Type: `SSH`, Port: `22`, Source: `My IP`.
    * **Rule 2 (Web App)**: Click **Add security group rule**.
        * Type: `Custom TCP`
        * Port Range: `5000`
        * Source: `Anywhere` (`0.0.0.0/0`)
7.  **Advanced details**: Scroll down and for **IAM instance profile**, select the **EC2 Instance Profile** you created.
8.  Click **Launch instance**.

<img width="1920" height="999" alt="image" src="https://github.com/user-attachments/assets/f73a0ba2-c7f3-4a38-bf3a-259b4eb0d11d" />

---

## 8. Connect to Your EC2 Instance

1.  From the EC2 dashboard, select your instance and copy its **Public IPv4 address**. Make sure the EC2 instance is in a **running state**, else the IPv4 address won't be visible.

2.  Open a terminal or SSH client and connect using your key pair:

    ```bash
    ssh -i /path/to/your-key-file.pem ec2-user@YOUR_PUBLIC_IP_ADDRESS
    ```

---

## 9. Set Up the Web Environment

The EC2 instance needs Python dependencies installed to run the web application. System updates and installing Flask and Boto3 provide functionality for running the server and interacting with AWS services. This step prepares the runtime environment for the dashboard.

**Approach:** Once connected via SSH, run the following commands to install the necessary software.

1.  **Update system packages**:
    ```bash
    sudo yum update -y
    ```
2.  **Install Python and Pip**:
    ```bash
    sudo yum install python3-pip -y
    ```
3.  **Install Python libraries (Flask & Boto3)**:
    ```bash
    pip3 install Flask boto3
    ```

---

## 10. Create and Configure the Web Application

The web application script fetches Athena data dynamically and displays results in a user-accessible interface. Updating configuration variables ensures the application connects to the correct AWS region, data catalog, and result location. This step personalizes the dashboard to your specific AWS setup.

**Approach:**

1.  Create the application file using the `nano` text editor:
    ```bash
    nano app.py
    ```
2.  Copy and paste your Python web application code (`EC2InstanceNANOapp.py`) into the editor.

3.  ‼️ **Important**: Update the placeholder variables at the top of the script (The below shown values are according to my setup):
    * `AWS_REGION`: us-east-2 (e.g., `us-east-1`).
    * `ATHENA_DATABASE`: orders_db (e.g., `orders_db`).
    * `S3_OUTPUT_LOCATION`: s3://cloudassignmentsudeepta/enriched/ (e.g., `s3://your-athena-results-bucket/`).

4.  Save the file and exit `nano` by pressing `Ctrl + X`, then `Y`, then `Enter`.

---

## 11. Run the App and View Your Dashboard! 🚀

Running the Flask application starts a web server hosted on your EC2 instance. Accessing the provided public URL in a browser loads the dashboard created from Athena query outputs. This final step completes the pipeline by visualizing processed data insights.

**Approach:**

1.  Execute the Python script to start the web server:
    ```bash
    python3 app.py
    ```
    You should see a message like `* Running on http://0.0.0.0:5000/`.

2.  Open a web browser and navigate to your instance's public IP address on port 5000:
    ```
    http://YOUR_PUBLIC_IP_ADDRESS:5000
    ```
    You should now see your Athena Order Dashboard like the one shown below!

    <img width="1470" height="837" alt="Screenshot 2025-11-10 at 11 51 31 AM" src="https://github.com/user-attachments/assets/a3e867d1-1c77-4847-82a3-4da57bf7210d" />

    <img width="1470" height="554" alt="Screenshot 2025-11-10 at 11 51 48 AM" src="https://github.com/user-attachments/assets/b9b2d649-138c-4ec5-96ed-c173e8445972" />

    <img width="1470" height="877" alt="Screenshot 2025-11-10 at 11 52 00 AM" src="https://github.com/user-attachments/assets/92947655-12a7-4501-b5a0-295d75366ffb" />

---

## Important Final Notes

* **Stopping the Server**: To stop the Flask application, return to your SSH terminal and press `Ctrl + C`.
* **Cost Management**: This setup uses free-tier services. To prevent unexpected charges, **stop or terminate your EC2 instance** from the AWS console when you are finished.

---

## Challenges faced and resolution

**Challenges faced**

After the **orders.csv** file was uploaded to the **raw/** folder of the **S3 bucket**, I was not able to see the processed file in the **processed/** folder of the bucket. Due to which I was not able to run the queries on the **Athena editor**.

**Resolution**

Once the **code** is written in the **code section of the Lambda function**, make sure to click on the **deploy** option; otherwise, the generation of the processed file won't be possible.

<img width="1920" height="999" alt="image" src="https://github.com/user-attachments/assets/12f3751b-f216-45a3-b0a4-682773fca264" />

---

