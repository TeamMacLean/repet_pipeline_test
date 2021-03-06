# REPET Pipeline Test

A complete tutorial for executing REPET pipeline steps is availabe [here](https://urgi.versailles.inra.fr/Tools/REPET/TEdenovo-tuto).

However, at many points, erros were encountered during testing. Here I am documenting any error I encountered while testing.

## Important note

You will need to run the pipeline from your project folder and These files should be in your project folder.

1) FASTA Sequence data that should be named as project e.g. DmelChr4.fa
2) ProfilesBankForREPET_Pfam27.0_GypsyDB.hmm
3) repbase20.05_ntSeq_cleaned_TE.fa
4) repbase20.05_aaSeq_cleaned_TE.fa
5) TEdenovo config file

## Step 1 genomic sequence preparation 

```TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 1```

This ran successfully

## Step 2 all by all alignment

```TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 2 -s Blaster```

This ran successfully

## Step 2 Structural detection

```TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 2 --struct```

This ran successfully

## Step 3 HSPs clustering

```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 3 -s Blaster -c Grouper
```

Failed to run with error message "grouper command not found". It was reported, installed and ran successfully

```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 3 -s Blaster -c Recon
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 3 -s Blaster -c Piler
```

These ran successfully.

## Step 3 clustering of structural detection 

```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 3 --struct
```

This ran successfully

## Step 4 Build consensus

```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 4 -s Blaster -c Grouper -m Map
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 4 -s Blaster -c Recon -m Map
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 4 -s Blaster -c Piler -m Map
```

These ran successfully

## Step 5 Consensus detect features

If you want to use only detection by similarity, you must have ran corresponding previous steps. Please launch the following command:

```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 5 -s Blaster -c GrpRecPil -m Map
```

This gives error. The error is:
File repbase20.05_aaSeq_cleaned_TE.fa not found. This file is in the project folder. However, the pipeline does not look in the project folder rather it looks in the newly created folder - DmelChr4_Blaster_GrpRecPil_Map_TEclassif/detectFeatures

Solution - I checked the script where it creates this folder and then added an extra line just after this code line to copy the file repbase20.05_aaSeq_cleaned_TE from project folder to this new folder.

I re-ran the command, the log showed it passed the line that generated previous error. Then there was a new error:

```
mysql_exceptions DatabaseError: ERROR can't execute CREATE INDEX iname on DmelChr4_sim_TR_set(name) after several tries.
```

This error is due to the limitation of 1000 bytes for the column "name" . This column was created in the database with schema VARCHAR(255). The specific error message while trying to create index :

```
Specified key was long; max key length is 1000 bytes
```

I tested creating index directly on mysql database and that constantly gave me the same error. After some online search, I found that I can place lower limit on the VARCHAR fields. I tried the mysql command

```
create index iname on DmelChr4_sim_TR_set(name(255))
```
This didn't work, gave me same error.

```
create index iname on DmelChr4_sim_TR_set(name(250))
```
This worked with no error.

So in the repet pipeline, I edit the code such that for any variable with VARCHAR(255), it creates the index with limit of 250.

Another variable to set was:

```
loose-local-infile = 1
```

You will need to be admin to change this in mysql config file. Thanks to Martin Page for editing the config file.

I re-ran the repet command and it worked finally.

If you want to use only structural detection, you must have ran corresponding previous steps (--struct option). Please launch the following command:

```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 5 -m Map --struct
```

This ran successfully

If you want to combine results from both means of detection and ran the corresponding previous steps, please launch the following command:
```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 5 -s Blaster -c GrpRecPil -m Map --struct 
```
This ran successfully

## Step 6 Consensus classification

If you want to use only detection by similarity, you must have ran corresponding previous steps. Please launch the following command:
```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 6 -s Blaster -c GrpRecPil -m Map
```

If you want to use only structural detection, you must have ran corresponding previous steps (--struct option). Please launch the following command: 
```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 6 -m Map --struct
```

If you want to combine results from both means of detection and ran the corresponding previous steps, please launch the following command:
```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 6 -s Blaster -c GrpRecPil -m Map --struct
```

## Step 7 Filtering 

```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 7 -s Blaster -c GrpRecPil -m Map
```

## Step 8 Consensus clustering
```
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 8 -s Blaster -c GrpRecPil -m Map -f Blastclust
TEdenovo.py -P DmelChr4 -C TEdenovo.cfg -S 8 -s Blaster -c GrpRecPil -m Map -f MCL
```
These ran successfully


# REPET job submission control

REPET submits thousands of jobs to the HPC. This crashes the job scheduler or becomes unfair HPC usages as all the resources get consumed by a single user. The number of jobs submission is hard coded. By default, the maximum number of jobs waiting is 50000 and the check interval is only 10 secs. This says that up to 50000 jobs can be at the waiting queue and check the queue every 10 secs. 

See the code below from the script (named TableJobAdaptor.py)

```
def submitJob(self, job, verbose=0, maxNbWaitingJobs=50000, checkInterval=10):
        if self.getJobStatus(job) in ["waiting", "running", "finished"]:
            sys.stderr.write("WARNING: job '%s' was already submitted\n" % job.jobname)
            sys.stderr.flush()
            self.cleanJobGroup(job.groupid)
            sys.exit(1)
        while self.getCountStatus(job.groupid, "waiting") > maxNbWaitingJobs:
            time.sleep(checkInterval)
 ```
 
This does not check how many jobs are running in the HPC.

I have added the following code, to check the running jobs.

```
        while self.getCountStatus(job.groupid, "running") > maxNbWaitingJobs:
            time.sleep(checkInterval)
```

The variable `maxNbWaitingJobs=50000`  and `checkInterval=10` have been changed to 10 and 600 respectively, to limit only 10 jobs (running or waiting) and check the queue status in every 10 mins (60 x 10). There are multiple scripts with these variables. In every script, where these variable occur, the values have been changed.

If required, these variables value can be changed to different values.

I have tested the REPET pipeline after the changes and I can see that the job submission is controlled.

