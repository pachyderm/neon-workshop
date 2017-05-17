# Workshop - Scalable, distributed deep learning with Python and Pachyderm

![alt tag](pipeline.jpg)

This workshop focuses on building a production scale machine learning pipeline with [Pachyderm](http://pachyderm.io/) that integrates [Nervana Neon](https://github.com/NervanaSystems/neon) training and inference.  In particular, this pipeline trains and utilizes a model that predicts the sentiment of movie reviews, based on data from IMDB.

The below documentation walks you through the deployment of the pipelines and emphasizes a few key features related to reproducibility, pipeline triggering, and provenance:

1. [Prepare a Python script and Docker image for training](README.md#1-prepare-a-python-script-and-docker-image-for-model-training)
2. [Prepare a Python script and Docker image for inference](README.md#2-prepare-a-python-script-and-docker-image-for-inference)
3. [Connect to your Pachyderm cluster](README.md#3-connect-to-your-pachyderm-cluster)
4. [Create the input "data repositories"](README.md#4-create-the-input-data-repositories)
5. [Commit the training data set into Pachyderm](README.md#5-commit-the-training-data-set-into-pachyderm)
6. [Create the training pipeline](README.md#6-create-the-training-pipeline)
7. [Commit input reviews](README.md#7-commit-input-reviews)
8. [Create the inference pipeline](README.md#8-create-the-inference-pipeline)
9. [Examine the results](README.md#9-examine-the-results)

Bonus:

10. [Parallelize the inference](README.md#10-parallelize-the-inference)
11. [Update the model training](README.md#11-update-the-model-training)
12. [Update the training data set](README.md#12-update-the-training-data-set)
13. [Examine pipeline provenance](README.md#13-examine-pipeline-provenance)

Finally, we provide some [Resources](README.md#resources) for you for further exploration.

## Prerequisites

- Ability to `ssh` into a remote machine.
- An IP for a remote machine (this should have been given to you at the beginning of the workshop).
- Access to this repository on GitHub (we will clone it later on the remote machine). 

## 1. Prepare a Python script and Docker image for model training

This part is actually pretty easy, because the team at Nervana has already created an [example python script](https://github.com/NervanaSystems/neon/blob/master/examples/imdb/train.py), `train.py`, for this very problem (sentiment analysis based on the IMDB data set).  You can read more about the implementation [here](http://neon.nervanasys.com/docs/latest/lstm.html).  Generally, `train.py` trains a recurrent neural network with Long-short Term Memory (LSTM) units.  You can read more about recurrent neural networks [here](http://www.wildml.com/2015/09/recurrent-neural-networks-tutorial-part-1-introduction-to-rnns/), and you can read more about LSTM units [here](http://colah.github.io/posts/2015-08-Understanding-LSTMs/).

For our purposes, we need to know how to run `train.py` on an input dataset, `labeledTrainData.tsv`:

```
python train.py -f labeledTrainData.tsv -e 2 -eval 1 -s imdb.p --vocab_file imdb.vocab
```

where `imdb.p` is a persistent representation of our model output from the script.

Each stage of our Pachyderm pipeline will be defined by a [Docker](https://www.docker.com/) image (among other things).  Thus, to use this training script as part of our ML pipeline we need to have a Docker image ready that includes and can run `train.py`.  We luck out here as well, because there is already [a public Docker image available on Docker Hub](https://hub.docker.com/r/kaixhin/neon/) with Neon and `train.py`.  

## 2. Prepare a Python script and Docker image for inference

We have to do a few custom things for inference.  The Neon example tutorial does include an inference script, but, as will be made clear soon, we actually want a Python script thatwill:

- take a directory as input
- walk over files in that directory, where the files include reviews
- infer the sentiment of each of the reviews
- output the inferred sentiment to a specified output directory

All of these steps are implemented in the included [auto_inference.py](inference/auto_inference.py) script.  We will run this script as follows:

```
python auto_inference.py --model_weights imdb.p --vocab_file imdb.vocab --review_files reviews --output_dir /out
```

To create a Docker image that will have this script available we have created a [corresponding Dockerfile](inference/Dockerfile).  For convenience, we have already built this image and uploaded it to Docker Hub [here](https://hub.docker.com/r/dwhitena/neon-inference/). 

## 3. Connect to your Pachyderm cluster  

You should have been given an IP for a remote machine at the beginning of the workshop.  The remote machine already has a Pachyderm cluster running locally and all of the command line tools we will be needing throughout the workshop.  To log into the remote machine, open and terminal and:

```
$ ssh pachrat@<remote machine IP>
```

You will be asked for a password, which you should also be given during the workshop.  To verify that everything is running correctly on the machine, you should be able to run the following with the corresponding response:

```
$ pachctl version
COMPONENT           VERSION             
pachctl             1.4.7-RC1           
pachd               1.4.7-RC1
```

## 4. Create the input data repositories 

On the Pachyderm cluster running in your remote machine, we will need to create the two input data repositories (for our training data and input movie reviews).  To do this run:

```
$ pachctl create-repo training
$ pachctl create-repo reviews
```

As a sanity check, we can list out the current repos, and you should see the two repos you just created:

```
$ pachctl list-repo
NAME                CREATED             SIZE                
reviews             2 seconds ago       0 B                 
training            8 seconds ago       0 B
```

## 5. Commit the training data set into pachyderm

We have our training data repository, but we haven't put our training data set into this repository yet.  You can get the training data set that we will be using via:

```
$ wget https://s3-us-west-2.amazonaws.com/wokshop-example-data/labeledTrainData.tsv
```

This `labeledTrainData.tsv` file include 750 labeled movie reviews sampled from a larger IMDB data set.  Here we are using a sample for illustrative purposes (so our examples run a little faster in the workshop), but the entire data set can be obtained [here](http://ai.stanford.edu/~amaas/data/sentiment/).

To get this data into Pachyderm, we run:

```
$ pachctl put-file training master -c -f labeledTrainData.tsv
```

Then, you should be able to see the following:

```
$ pachctl list-repo
NAME                CREATED             SIZE                
training            6 minutes ago       977 KiB             
reviews             6 minutes ago       0 B                 
$ pachctl list-file training master
NAME                   TYPE                SIZE                
labeledTrainData.tsv   file                977 KiB
```

## 6. Create the training pipeline

Next, we can create the `model` pipeline stage to process the data in the training repository. To do this, we just need to provide Pachyderm with [a JSON pipeline specification](train.json) that tells Pachyderm how to process the data.  You can copy `train.json` from this repository or clone the whole repository to your remote machine.  After you have `infer.json`, creating our `model` training pipeline is as easy as:

```
$ pachctl create-pipeline -f train.json
```

Immediately you will notice that Pachyderm has kicked off a job to perform the model training:

```
$ pachctl list-job
ID                                   OUTPUT COMMIT STARTED       DURATION RESTART PROGRESS STATE            
82682496-5b41-45a0-96f7-175812274dd8 model/-       2 seconds ago -        0       0 / 1    running
```

This job should run for about 3-8 minutes.  In the mean time, we will address any questions that might have come up, help any users with issues they are experiences, and talk a bit more about ML workflows in Pachyderm.

After your model has successfully been trained, you should see:

```
$ pachctl list-job
ID                                   OUTPUT COMMIT                          STARTED        DURATION  RESTART PROGRESS STATE            
82682496-5b41-45a0-96f7-175812274dd8 model/5559fce59d9748b883ff8af6bc60eeb1 11 minutes ago 3 minutes 0       1 / 1    success
$ pachctl list-repo
NAME                CREATED             SIZE                
reviews             21 minutes ago      0 B            
model               10 minutes ago      20.93 MiB           
training            21 minutes ago      977 KiB             
$ pachctl list-file model master
NAME                TYPE                SIZE                
imdb.p              file                20.54 MiB           
imdb.vocab          file                393.2 KiB
```

## 7. Commit input reviews

Great! We now have a trained model that will infer the sentiment of movie reviews.  Let's commit some movie reviews into Pachyderm that we would like to run through the sentiment analysis.  We have a couple examples under [test](test).  Feel free to use these, find your own, or even write your own review.  To commit our samples (assuming you have cloned this repo on the remote machine), you can run:

```
$ cd test
$ pachctl put-file reviews master -c -r -f .
```

You should then see:

```
$ pachctl list-file reviews master
NAME                TYPE                SIZE                
1.txt               file                770 B               
2.txt               file                897 B
```

## 8. Create the inference pipeline

We have another JSON blob, [infer.json](infer.json), that will tell Pachyderm how to perform the processing for the inference stage.  This is similar to our last JSON specification except, in this case, we have two input repositories (the `reviews` and the `model`) and we are using a different Docker image that contains `auto_inference.py`.  To create the inference stage, we simply run:

```
$ pachctl create-pipeline -f infer.json
```

This will immediately kick off an inference job, because we have committed unprocessed reviews into the `reviews` repo.  The results will then be versioned in a corresponding `inference` data repository:

```
$ pachctl list-job
ID                                   OUTPUT COMMIT                          STARTED           DURATION  RESTART PROGRESS STATE            
40440de9-542f-4f32-80ce-bc52b5222424 inference/-                            3 seconds ago     -         0       0 / 2    running 
82682496-5b41-45a0-96f7-175812274dd8 model/5559fce59d9748b883ff8af6bc60eeb1 About an hour ago 3 minutes 0       1 / 1    success 
$ pachctl list-job
ID                                   OUTPUT COMMIT                              STARTED            DURATION   RESTART PROGRESS STATE            
40440de9-542f-4f32-80ce-bc52b5222424 inference/cd548a0b651b4c188b5c66ab0f12ae96 About a minute ago 12 seconds 0       2 / 2    success 
82682496-5b41-45a0-96f7-175812274dd8 model/5559fce59d9748b883ff8af6bc60eeb1     About an hour ago  3 minutes  0       1 / 1    success 
$ pachctl list-repo
NAME                CREATED              SIZE                
inference           About a minute ago   70 B                
reviews             2 hours ago          1.628 KiB           
model               About an hour ago    20.93 MiB           
training            2 hours ago          977 KiB
```

## 9. Examine the results

We have created results from the inference, but how do we examine those results?  There are multiple ways, but an easy way is to just "get" the specific files out of Pachyderm's data versioning:

```
$ pachctl list-file inference master
NAME                TYPE                SIZE                
1.txt               file                35 B                
2.txt               file                35 B                
$ pachctl get-file inference master 1.txt
Pred - [[ 0.50981182  0.49018815]]
$ pachctl get-file inference master 2.txt
Pred - [[ 0.53010267  0.46989727]]
```

Here we can see that each result file contains two probabilities corresponding to postive and negative sentiment, respectively.

## Bonus exercises

You may not get to all of these bonus exercises during the workshop time, but you can perform these and all of the above steps any time you like with a [simple local Pachyderm install](http://docs.pachyderm.io/en/latest/getting_started/local_installation.html).  You can spin up this local version of Pachyderm is just a few commands and experiment with this, [other Pachyderm examples](http://docs.pachyderm.io/en/latest/examples/readme.html), and/or your own pipelines.

### 10. Parallelize the inference

You may have noticed that our pipeline specs included a `parallelism_spec` field.  This tells Pachyderm how to parallelize a particular pipeline stage.  Let's say that in production we start receiving a huge number of movie reviews, and we need to keep up with our sentiment analysis.  In particular, let's say we want to spin up 10 inference workers to perform sentiment analysis in parallel.

This actually doesn't require any change to our code.  We can simply change our `parallelism_spec` to:

```
  "parallelism_spec": {
    "strategy": "CONSTANT",
    "constant": "10"
  },
```

Pachyderm will then spin up 10 inference workers, each running our same `auto_inference.py` script, to perform inference in parallel.  This can be confirmed by updating our pipeline and then examining the cluster:

```
$ vim infer.json 
$ pachctl update-pipeline -f infer.json 
$ kubectl get all
NAME                             READY     STATUS        RESTARTS   AGE
po/etcd-4197107720-xl44v         1/1       Running       0          2h
po/pachd-3548222380-6nx8j        1/1       Running       0          2h
po/pipeline-inference-v1-k5vzq   2/2       Terminating   0          14m
po/pipeline-inference-v2-0c8g3   0/2       Pending       0          3s
po/pipeline-inference-v2-1bstd   0/2       Init:0/1      0          3s
po/pipeline-inference-v2-2jm3f   0/2       Init:0/1      0          3s
po/pipeline-inference-v2-30mc3   0/2       Pending       0          3s
po/pipeline-inference-v2-8sxnr   0/2       Init:0/1      0          3s
po/pipeline-inference-v2-cspvv   0/2       Init:0/1      0          3s
po/pipeline-inference-v2-ks539   0/2       Pending       0          3s
po/pipeline-inference-v2-t8lgz   0/2       Init:0/1      0          3s
po/pipeline-inference-v2-x7xdv   0/2       Init:0/1      0          3s
po/pipeline-inference-v2-z3grg   0/2       Init:0/1      0          3s
po/pipeline-model-v1-ps9sl       2/2       Running       0          2h

NAME                       DESIRED   CURRENT   READY     AGE
rc/pipeline-inference-v2   10        10        0         3s
rc/pipeline-model-v1       1         1         1         2h

NAME                        CLUSTER-IP      EXTERNAL-IP   PORT(S)                       AGE
svc/etcd                    10.96.190.86    <nodes>       2379:32379/TCP                2h
svc/kubernetes              10.96.0.1       <none>        443/TCP                       2h
svc/pachd                   10.103.153.25   <nodes>       650:30650/TCP,651:30651/TCP   2h
svc/pipeline-inference-v2   10.101.50.128   <none>        80/TCP                        3s
svc/pipeline-model-v1       10.96.86.16     <none>        80/TCP                        2h

NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/etcd    1         1         1            1           2h
deploy/pachd   1         1         1            1           2h

NAME                  DESIRED   CURRENT   READY     AGE
rs/etcd-4197107720    1         1         1         2h
rs/pachd-3548222380   1         1         1         2h
$ kubectl get all
NAME                             READY     STATUS        RESTARTS   AGE
po/etcd-4197107720-xl44v         1/1       Running       0          2h
po/pachd-3548222380-6nx8j        1/1       Running       0          2h
po/pipeline-inference-v2-0c8g3   2/2       Running       0          31s
po/pipeline-inference-v2-1bstd   2/2       Running       0          31s
po/pipeline-inference-v2-2jm3f   2/2       Running       0          31s
po/pipeline-inference-v2-30mc3   2/2       Running       0          31s
po/pipeline-inference-v2-8sxnr   2/2       Running       0          31s
po/pipeline-inference-v2-cspvv   2/2       Running       0          31s
po/pipeline-inference-v2-ks539   2/2       Running       0          31s
po/pipeline-inference-v2-t8lgz   2/2       Running       0          31s
po/pipeline-inference-v2-x7xdv   2/2       Running       0          31s
po/pipeline-inference-v2-z3grg   2/2       Running       0          31s
po/pipeline-model-v1-ps9sl       2/2       Running       0          2h

NAME                       DESIRED   CURRENT   READY     AGE
rc/pipeline-inference-v2   10        10        10        31s
rc/pipeline-model-v1       1         1         1         2h

NAME                        CLUSTER-IP      EXTERNAL-IP   PORT(S)                       AGE
svc/etcd                    10.96.190.86    <nodes>       2379:32379/TCP                2h
svc/kubernetes              10.96.0.1       <none>        443/TCP                       2h
svc/pachd                   10.103.153.25   <nodes>       650:30650/TCP,651:30651/TCP   2h
svc/pipeline-inference-v2   10.101.50.128   <none>        80/TCP                        31s
svc/pipeline-model-v1       10.96.86.16     <none>        80/TCP                        2h

NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/etcd    1         1         1            1           2h
deploy/pachd   1         1         1            1           2h

NAME                  DESIRED   CURRENT   READY     AGE
rs/etcd-4197107720    1         1         1         2h
rs/pachd-3548222380   1         1         1         2h
```

### 11. Update the model training

**Note** - This exercise increases the training time of our model, so you might be waiting for 5+ minutes for the model to re-train (maybe up to 10-15 minutes).  If you don't want to wait for this amount of time during the workshop, you could try step 12, which will take less time.

You might have noticed that we only used one "epoch" in our model training the first time around.  This is probably not enough in general.  As such, you can change this to two, for example, by modifying `train.json`:

```
    "cmd": [ 
	"python", 
	"examples/imdb/train.py", 
	"-f", 
	"/pfs/training/labeledTrainData.tsv", 
	"-e", 
	"2", 
	"-eval", 
	"1", 
	"-s", 
	"/pfs/out/imdb.p", 
	"--vocab_file", 
	"/pfs/out/imdb.vocab" 
    ]
```

Once you modify the spec, you can update the pipeline by running:

```
$ pachctl update-pipeline -f train.json
```

Pachyderm will then automatically kick off a new job to retrain our model with the updated parameters:

```
$ pachctl list-job
ID                                   OUTPUT COMMIT                              STARTED        DURATION           RESTART PROGRESS STATE             
31095b41-d32e-4f96-b057-5a47df695e04 model/-                                    2 minutes ago  -                  1       0 / 1    running  
40440de9-542f-4f32-80ce-bc52b5222424 inference/cd548a0b651b4c188b5c66ab0f12ae96 47 minutes ago 12 seconds         0       2 / 2    success  
82682496-5b41-45a0-96f7-175812274dd8 model/5559fce59d9748b883ff8af6bc60eeb1     2 hours ago    3 minutes          0       1 / 1    success 
```

Not only that, once the model is retrained, Pachyderm see the new model and updates our inferences with the latest version of the model:

```
$ pachctl list-job
ID                                   OUTPUT COMMIT                              STARTED        DURATION           RESTART PROGRESS STATE            
681ccee7-24d8-4d78-8ca7-81c539fe973a inference/ce133b12400a46248a2b46917a6ce9f2 3 minutes ago  1 seconds          0       2 / 2    success 
31095b41-d32e-4f96-b057-5a47df695e04 model/da368d8220d64db2924e73c9d8e428c7     8 minutes ago  4 minutes          1       1 / 1    success 
40440de9-542f-4f32-80ce-bc52b5222424 inference/cd548a0b651b4c188b5c66ab0f12ae96 53 minutes ago 12 seconds         0       2 / 2    success 
82682496-5b41-45a0-96f7-175812274dd8 model/5559fce59d9748b883ff8af6bc60eeb1     2 hours ago    3 minutes          0       1 / 1    success
```

### 12. Update the training data set

Let's say that one or more observations in our training data set were corrupt or unwanted.  Thus, we want to update our training data set.  To simulate this, go ahead and open up `labeledTrainData.tsv` and remove a couple of the reviews (i.e., the non-header rows).  Then, let's replace our training set:

```
$ pachctl start-commit training master
9cc070dadc344150ac4ceef2f0758509
$ pachctl delete-file training 9cc070dadc344150ac4ceef2f0758509 labeledTrainData.tsv 
$ pachctl put-file training 9cc070dadc344150ac4ceef2f0758509 -f labeledTrainData.tsv 
$ pachctl finish-commit training 9cc070dadc344150ac4ceef2f0758509
```

Immediately, Pachyderm "knows" that the data has been updated, and it starts a new job to update the model and inferences:

```
$ pachctl list-job
ID                                   OUTPUT COMMIT                              STARTED        DURATION   RESTART PROGRESS STATE            
b76f5da2-13ab-4696-9747-6e3555c244bf model/-                                    52 seconds ago -          0       0 / 1    running 
40440de9-542f-4f32-80ce-bc52b5222424 inference/cd548a0b651b4c188b5c66ab0f12ae96 20 minutes ago 12 seconds 0       2 / 2    success 
82682496-5b41-45a0-96f7-175812274dd8 model/5559fce59d9748b883ff8af6bc60eeb1     2 hours ago    3 minutes  0       1 / 1    success
```

Not only that, when the new model has been produced, Pachyderm "knows" that there is a new model and updates the previously inferred sentiments:

```
$ pachctl list-job
ID                                   OUTPUT COMMIT                              STARTED        DURATION           RESTART PROGRESS STATE            
22309e93-098b-4721-ae6e-d69de6c59dac inference/18d5502ec45f489cb90eaea133a057f2 18 seconds ago Less than a second 0       2 / 2    success 
b76f5da2-13ab-4696-9747-6e3555c244bf model/14fa56e673404b3eb9e12157cee77a9c     2 minutes ago  2 minutes          0       1 / 1    success 
40440de9-542f-4f32-80ce-bc52b5222424 inference/cd548a0b651b4c188b5c66ab0f12ae96 21 minutes ago 12 seconds         0       2 / 2    success 
82682496-5b41-45a0-96f7-175812274dd8 model/5559fce59d9748b883ff8af6bc60eeb1     2 hours ago    3 minutes          0       1 / 1    success
```

### 13. Examine pipeline provenance

Let's say that we have updated our model or training set in one of the above scenarios (step 11 or 12).  Now we have multiple inferences that were made with different models and/or training data sets.  How can we know which results came from which specific models and/or training data sets?  This is called "provenance," and Pachyderm gives it to you out of the box.  

Suppose we have run the following jobs:

```
$ pachctl list-job
ID                                   OUTPUT COMMIT                              STARTED        DURATION           RESTART PROGRESS STATE            
22309e93-098b-4721-ae6e-d69de6c59dac inference/18d5502ec45f489cb90eaea133a057f2 18 seconds ago Less than a second 0       2 / 2    success 
b76f5da2-13ab-4696-9747-6e3555c244bf model/14fa56e673404b3eb9e12157cee77a9c     2 minutes ago  2 minutes          0       1 / 1    success 
2e2ba0ec-1a99-4b15-97c0-32a38aa910fb inference/37c1247e40bc44de871b63d20472de4d 7 minutes ago  21 seconds         0       2 / 2    success 
40440de9-542f-4f32-80ce-bc52b5222424 inference/cd548a0b651b4c188b5c66ab0f12ae96 21 minutes ago 12 seconds         0       2 / 2    success 
82682496-5b41-45a0-96f7-175812274dd8 model/5559fce59d9748b883ff8af6bc60eeb1     2 hours ago    3 minutes          0       1 / 1    success
```

If we want to know which model and training data set was used for the latest inference, commit id `18d5502ec45f489cb90eaea133a057f2`, we just need to inspect the particular commit:

```
$ pachctl inspect-commit inference 18d5502ec45f489cb90eaea133a057f2
Commit: inference/18d5502ec45f489cb90eaea133a057f2
Parent: 37c1247e40bc44de871b63d20472de4d 
Started: 13 minutes ago
Finished: 13 minutes ago 
Size: 70 B
Provenance:  training/9cc070dadc344150ac4ceef2f0758509  model/14fa56e673404b3eb9e12157cee77a9c  reviews/ee3c342ab66e4ea887e5a3b994ac2f7d
```

The `Provenance` tells us exactly which model and training set was used (along with which commit to reviews triggered the sentiment analysis).  For example, if we wanted to see the exact model used, we would just need to reference commit `14fa56e673404b3eb9e12157cee77a9c` to the `model` repo:

```
$ pachctl list-file model 14fa56e673404b3eb9e12157cee77a9c
NAME                TYPE                SIZE                
imdb.p              file                20.54 MiB           
imdb.vocab          file                392.6 KiB
```

We could get this model to examine it, rerun it, revert to a different model, etc.

## Resources

Pachyderm:

- Join the [Pachyderm Slack team](http://slack.pachyderm.io/) to ask questions, get help, and talk about production deploys.
- Follow [Pachyderm on Twitter](https://twitter.com/pachydermIO), 
- Find [Pachyderm on GitHub](https://github.com/pachyderm/pachyderm), and
- [Spin up Pachyderm](http://docs.pachyderm.io/en/latest/getting_started/getting_started.html) in just a few commands to try this and [other examples](http://docs.pachyderm.io/en/latest/examples/readme.html) locally.

Nervana Neon:

- Check out the [Neon docs](http://neon.nervanasys.com/docs/latest/),
- Try out one of the [tutorials](http://neon.nervanasys.com/docs/latest/tutorials.html), and 
- Follow [Nervana on Twitter](https://twitter.com/nervanasys).

