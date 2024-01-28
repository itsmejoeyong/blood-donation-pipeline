# blood-donation-pipeline
A pipeline to pull & process data from github into a mysql database, visualized using LookerStudio.

```
/project-root
| -- /data         # contains [raw/staging/cleaned] directories for processed data
| -- /logs         # contains application logs
| -- misc          # contains misc scipts including dags, (base) docker-compose, lambda functions
| -- src           # contains sql code & notifications source files
| -- src/etl       # contains [extract/transform/load] source files 
| -- tests         # contains pytest tests
```

```
/home/ubuntu
| -- airflow                        # contains the dockerized airflow instance
| -- blood-donation-pipeline        # contains the github repo
```

### Workflow
1. EventBridge + Lambda function turns on the ec2 instance, kick-starting Airflow, and eventually executing the script via a bash command (explain later).
   > I do this so i don't have to pay as much
2. Prepare the target database by creating tables, indexes, & procedures in advance.
3. pulls the source data from the [github](https://github.com/MoH-Malaysia/data-darah-public) raw files & (parquet)[https://dub.sh/ds-data-granular] link given into [data/raw] directory
4. Filters latest data into [data/staging] directory by extracting the latest date column in database, then slicing the the retrieved raw files.
   > This is done as I notice data is not updated everyday, so there is a chance that we may miss some data if we only slice "yesterday's data".
5. Validates the data to ensure the columns have no outliers (str in date/str in int) into the [data/cleaned] directory
   1. - The parquet file goes through addtional processing and transformation, as sql database runs out of [/tmp] to do aggregation
   2. - The parquet files are then directly uploaded to the database (on the ec2)
6. The rest of the aggregated ([csv files](https://github.com/MoH-Malaysia/data-darah-public)) data are then uploaded to the database (on the ec2)
7. A Telegram message will be sent to a groupchat via bot that the process has been completed.
8. During the brief time window when the ec2 instance is active, LookerStudio will cache the data.
   > LookerStudio doesn't normally cache the data unlike PowerBI, therefore I setup the data source (to the database) to refresh every 15 minutes, within that 30 minute window when the ec2 instance is active, a separate LookerStudio data source (extract data, aka caching data) will capture the data.

### Notes
- I originally started development in a github repository, then porting to Airflow, should've started backwards to make use of dags & tasks.
- Storing of data could've be on s3, database could've been bigquery, but due to time and the size of the project (being pretty small) I've sticked with the classic & simple monolithic approach.
- I cannot seem to run python-telegram-bot in a venv using Airflow, therefore I've used requests.get() instead. Barbaric but it works.
  > It works when manually running via the terminal/bash script:
  cd /home/ubuntu/blood-donation-pipeline
  source venv/bin/activate
  python main.py
  >
  > However when ran via an airflow BashOperator, it cannot import the python-telegram-package.
- There is one big limitation, just because the bot has notified the group, doesn't necessarily mean the data is updated, it depends on when the underlying data source has been refresh, because of this the data may be delayed by up to 30 minutes from the notification sent.
- the docker-compose.yaml is only the base one I've used with a slight modifications so that it can use the github repo as they're both in separate directories on ec2 in #volumes section
- I wrap the sql preperation queries in a try block, as I've had certain warning's on mysql get flagged as exceptions and stops the code.

### SQL procedures
1. [Query](https://github.com/JoeYong01/blood-donation-pipeline/blob/main/misc/sql.py) for question 2 
> This cannot be run due the limited /tmp size that mysql has, and i couldn't get it to write to a custom directory, therefore I've decided to do the transformation logic directly on the airflow vm.