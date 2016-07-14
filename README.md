## Quick Ad-Hoc Spark Cluster on DigitalOcean in 10 minutes

### Prereqs

##### Tools

You need python, node, and ansible on your system.

```shell
brew install ansible
```

##### Digital Ocean account and api token

Sign up for an account and create an api token.

##### SSH Keys

You need an ssh key setup on digitalocean. You can create a key and place it in `~/keys/`

```shell
ssh-keygen -t rsa -f do
```

You need to add the contents of ~/keys/do.pub to digital ocean

* Copy file contents into clipboard: `pbcopy < ~/keys/do.pub`
* Add key: Your Digital Ocean profile > Security > SSH Keys
* Give it the name, `key-<IP>`, which corresponds to your public IP address.

### AutoSpark

Get AutoSpark, which will help you create a new spark cluster.

```shell
git clone https://github.com/alt-code/AutoSpark
cd AutoSpark/driver
npm install
```

##### Create a cluster

```
node autospark-cluster-launcher.js
# Fill in the blanks...
prompt: provider:  digitalocean
prompt: digitalocean_token: 66....
prompt: size:  large
prompt: name:  sparkparnin
prompt: ssh_private_key_path:  /Users/cjparnin/keys/do
prompt: ssh_public_key_path:  /Users/cjparnin/keys/do.pub
(needs your password for sudo)
```

You should see something like...

```
Data networks: 45.55.232.179
Data networks: 45.55.207.49
Info: IP Address of Master 104.236.17.68
Info: Spark Master at spark://104.236.17.68:7077
```

**Control-C program to quit...**

##### Setup spark with ansible

Autospark will do this for you, but it is nice to see the manual part:

```
cd ../Ansible/playbooks
# Configure the master node
./master.sh 
# Configure the worker nodes
./slave.sh
```

If you see issues connecting to slave nodes (seems to ask for password, then try this):

```
sudo vi /etc/ssh/ssh_config
StrictHostKeyChecking no
```

##### View your cluster

In a browser, go to http://104.236.17.68:8080 make sure you can see a cluster with some workers.

![spark cluster](https://cloud.githubusercontent.com/assets/742934/16849830/4c76afaa-49ca-11e6-83fc-4c0ee19a6a11.png)

### Run a simple job

SSH into your master machine.

```
ssh -i ~/keys/do root@104.236.17.68
cd /spark/spark_latest/examples/src/main/python
```

Submit a job to cluster
```
# Set ip for locally submitted jobs
export SPARK_LOCAL_IP=104.236.17.68
# Submit job to cluster
/spark/spark_latest/bin/spark-submit --master spark://104.236.17.68:7077 pi.py  
```

Output should look something like:
```
16/07/14 12:58:12 INFO DAGScheduler: Job 0 finished: reduce at /spark/spark_latest/examples/src/main/python/pi.py:39, took 7.137977 s
Pi is roughly 3.140360
```

### Errors

If you see:
```
TaskSchedulerImpl: Initial job has not accepted any resources; check your cluster UI to ensure that workers are registered and have sufficient resources
```

It means you're spark master cannot connect to workers, or they have refused jobs because they do not have enough memory.


### REST

You can send a rest request to cluster to submit jobs.
Now from your computer, try sending a REST request with curl.

```shell
curl -X POST http://45.55.207.49:6066/v1/submissions/create --header "Content-Type:application/json;charset=UTF-8" --data '{
      "action" : "CreateSubmissionRequest",
      "appArgs" : [ "/spark/spark_latest/examples/src/main/python/pi.py" ],
      "appResource" : "file:/spark/spark_latest/examples/src/main/python/pi.py",
      "clientSparkVersion" : "1.6.2",
      "environmentVariables" : {
        "SPARK_ENV_LOADED" : "1",
        "SPARK_MASTER_IP" : "45.55.207.49"
      },
      "mainClass" : "org.apache.spark.deploy.SparkSubmit",
      "sparkProperties" : {
        "spark.app.name" : "MyJob",
        "spark.eventLog.enabled": "false",
        "spark.submit.deployMode" : "client",        
        "spark.master" : "spark://45.55.207.49:7077"
      }
    }'
```

You can see the results in Completed Drivers, click worker link, scroll down to finished drivers, and click stdout log. (It may say state failed even though job completed ok).

### Benefits

Running LDA on wikipedia. Something that couldn't run on a single machine, could be done in about an hour!

##### 8GB dataset
| Parameters     | 15 nodes | 30 nodes | 45 nodes | 
| ------------- |:-------------:|:-------------:|:-------------:|
| Total Time | Exited after 1.3 hour | 2.1 hour | 1.1 hour |
| Total used memory     | Error | 60.9 GB | 61 GB |
| Memory usage per node | Error | 2100 MB | 1300 MB |
| Input per node | Error | 9 GB | 7 GB | 

### Shutting Down

Stop paying bills

```shell
node autospark-teardown.js
```
