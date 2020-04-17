# 6. Bayes Classifier on Cloud Dataproc

To repeat the steps in this chapter, follow these steps.

### Catch up from previous chapters if necessary
If you didn't go through Chapters 2-5, the simplest way to catch up is to copy data from my bucket:
* Go to the 02_ingest folder of the repo, run the program ./ingest_from_crsbucket.sh and specify your bucket name.
* Go to the 04_streaming folder of the repo, run the program ./ingest_from_crsbucket.sh and specify your bucket name.
* Create a dataset named "flights" in BigQuery by typing:
	```
	bq mk flights
	```
* Go to the 05_bqdatalab folder of the repo, run the script to load data into BigQuery:
	```
	bash load_into_bq.sh <BUCKET-NAME>
	```
* In BigQuery, run this query and save the results as a table named trainday
	```
	  #standardsql
	SELECT
	  FL_DATE,
	  IF(MOD(ABS(FARM_FINGERPRINT(CAST(FL_DATE AS STRING))), 100) < 70, 'True', 'False') AS is_train_day
	FROM (
	  SELECT
	    DISTINCT(FL_DATE) AS FL_DATE
	  FROM
	    `flights.tzcorr`)
	ORDER BY
	  FL_DATE
	```

### Create Dataproc cluster
In CloudShell:
* Clone the repository if you haven't already done so:
    ```
    git clone https://github.com/GoogleCloudPlatform/data-science-on-gcp
    ```
* Change to the directory for this chapter:
    ```
    cd data-science-on-gcp/06_dataproc
    ```
* Create the Dataproc cluster to run jobs on, specifying the name of your bucket and a 
  zone in the region that the bucket is in. (You created this bucket in Chapter 2)
   ```
    ./create_cluster.sh <BUCKET-NAME>  <COMPUTE-ZONE>
    ```
*Note:* Make sure that the compute zone is in the same region as the bucket, otherwise you will incur network egress charges.

### Quantization using Spark SQL
On your <em>local</em> machine (<b>i.e. not on GCP</b>):
* Install the <a href="https://cloud.google.com/sdk/downloads">gcloud SDK</a> if you haven't already done so:
* Create a SSH tunnel to your Dataproc cluster (change the zone appr
    ```
    gcloud compute ssh  --zone=us-central1-a  \
          --ssh-flag="-D 1080" --ssh-flag="-N" --ssh-flag="-n" \
          ch6cluster-m
    ```
   + To check result of instances
   ```
   gcloud compute instances list
   ```

* Start a new Chrome browser window (you can leave your other Chrome windows running).
  Specify a non-existent directory (instead of /tmp/junk) and change the path to Chrome
  appropriately.
    ```
    rm -rf /tmp/junk
    /usr/bin/chrome \
      --proxy-server="socks5://localhost:1080" \
      --host-resolver-rules="MAP * 0.0.0.0 , EXCLUDE localhost" \
      --user-data-dir=/tmp/junk
    ```
    For example, if you are on Mac OS-X, the path to Chrome is:
    ```
    /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome 
    ```

    ```
    rm -rf /Users/xunliu/gcp_tmp
    /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --proxy-server="socks5://localhost:1080" \
  --host-resolver-rules="MAP * 0.0.0.0 , EXCLUDE localhost" \
  --user-data-dir=/Users/xunliu/gcp_tmp
    ```

* In the new Chrome window, navigate to ```http://ch6cluster-m:8080/```, after making sure to allow
  outgoing http traffic from your local machine to the Dataproc cluster.
    + Default port number of `jupyter` is 8124
    + The link should be `http://ch6cluster-m:8124/gateway/default/jupyter/`
    + 

* In the new browser window, copy-and-paste cells from <a href="quantization.ipynb">quantization.ipynb</a>.
  Make sure to set the appropriate values in the cell containing the PROJECT, BUCKET, and REGION.
 
* [optional] make the changes suggested in the notebook to run on the full dataset.  Note that you might have to
  reduce numbers to fit into your quota.
  
### Bayes Classification using Pig
* SSH into the master node of the cluster by going to the GCP console
* Clone the repository:
    ```
    git clone git clone https://github.com/GoogleCloudPlatform/data-science-on-gcp
    ```
* Change to the directory for this chapter:
    ```
    cd data-science-on-gcp/06_dataproc
    ```
* Change the bucket name in the Pig script:
    ```
    sed 's/cloud-training-demos-ml/YOUR_BUCKET_NAME/g' bayes_final.pig > /tmp/bayes.pig
    ```
* Submit Pig job to do Bayes classification (it will take a while to complete):
    ```
    gcloud dataproc jobs submit pig \
         --cluster ch6cluster --file /tmp/bayes.pig
    ```

### Delete the cluster
* Delete the cluster either from the GCP web console or by typing in CloudShell, ```./delete_cluster.sh```
  ```
  gcloud dataproc clusters delete --region=us-west2 ch6cluster
  ```



### Errors 

#### Failed to create instance for free account

```
Enabling service [dataproc.googleapis.com] on project [689471184059]...
Operation "operations/acf.d8d9a973-c1b5-4e6c-9fbc-fbe58b977dd2" finished successfully.
ERROR: (gcloud.beta.dataproc.clusters.create) INVALID_ARGUMENT: Multiple validation errors:
 - Insufficient 'CPUS' quota. Requested 12.0, available 8.0.
 - Insufficient 'DISKS_TOTAL_GB' quota. Requested 3000.0, available 1848.0.
 - This request exceeds CPU quota. Some things to try: request fewer workers (a minimum of 2 is required), use smaller master and/or worker machine types (such as n1-standard-2).

```
solution: downgrade resources
```
gcloud beta dataproc clusters create \
   --num-workers=2 \
   --scopes=cloud-platform \
   --worker-machine-type=n1-standard-4 \
   --worker-boot-disk-size=500GB \
   --master-machine-type=n1-standard-4 \
   --master-boot-disk-size=500GB \
   --image-version=1.4 \
   --enable-component-gateway \
   --optional-components=ANACONDA,JUPYTER \
   --region=us-west2
   --zone=$ZONE \
   --initialization-actions=$INSTALL \
   ch6cluster

```

#### Connection failed

```
➜  ~ gcloud compute ssh  --zone=us-west2-a --ssh-flag="-D 1080" --ssh-flag="-N" --ssh-flag="-n" ch6cluster-m
channel 6: open failed: administratively prohibited: open failed
channel 7: open failed: administratively prohibited: open failed
channel 8: open failed: administratively prohibited: open failed
channel 12: open failed: connect failed: Connection refused
channel 13: open failed: connect failed: Connection refused
channel 12: open failed: connect failed: Connection refused
channel 13: open failed: connect failed: Connection refused
```
- Change `AllowTcpForwarding`
- https://serverfault.com/questions/535412/how-to-solve-the-open-failed-administratively-prohibited-open-failed-when-us
- https://unix.stackexchange.com/questions/14160/ssh-tunneling-error-channel-1-open-failed-administratively-prohibited-open
