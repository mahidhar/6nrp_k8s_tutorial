# 6NRP Kubernetes Tutorial

Typical usage workflow\
Hands on session

## Use an existing application container to run jobs

Lets use an existing LAMMPS (a molecular dynamics code) container from dockerhub. 

###### lammps.yaml:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: lammps-<username>
spec:
  template:
    spec:
      volumes:
          - name: scratch
            emptyDir: {}
      containers:
      - name: test
        image: lammps/lammps:patch_7Jan2022_rockylinux8_openmpi_py3
        command: ["/bin/bash", "-c"]
        args:
        - >-
            cd /scratch;
            curl -O https://www.lammps.org/bench/inputs/in.lj.txt ;
            export OMP_NUM_THREADS=1;
            lmp_serial < in.lj.txt ;
            mpirun --oversubscribe -np 4 lmp_mpi < in.lj.txt;
        volumeMounts:
            - name: scratch
              mountPath: /scratch
        resources:
          limits:
            memory: 16Gi
            cpu: "4"
            ephemeral-storage: 10Gi
          requests:
            memory: 16Gi
            cpu: "4"
            ephemeral-storage: 10Gi
      restartPolicy: Never
```
Lets run this simple application test:

```
kubectl apply -f lammps.yaml
```
Now lets check the output:
```
kubectl logs lammps-mahidhar-cj25r
```

Delete the job once we have checked output:
```
kubectl delete -f lammps.yaml
```

## Download input data; Download, build, and run scientific code

Let's now work through a scientific application workflow. In this example we will: (a) Download the input data from an external source and put it in the ephemeral storage; (b) Download and build the scientific code; and (c) Run the benchmark.

The container we have chosen is a standard gcc container from dockerhub. The application we will use is the Metagenome Annotation workflow component benchmark
from the NERSC-10 Benchmark Suite (https://gitlab.com/NERSC/N10-benchmarks/hmmsearch-benchmark). This benchmark is based on the computation workflow of the Joint Genome Institute's (JGI)'s Integrated Microbial Genomes project (IMG). IMG's scientific objective is to analyze, annotate and distribute microbial genomes and microbiome datasets.

The application hmmsearch is one program within the HMMER biosequence analysis package that uses a profile Hidden Markov Models (HMM) algorithm to perform a search for matches between input sequences and a reference database of sequence profiles. 

Lets start by spinning up a pod with the gcc environment with a sleep of 3600s. 
We will walk through the benchmark steps manually after the pod starts.

###### scienceapp.yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scienceapp-<username>
spec:
      volumes:
          - name: scratch
            emptyDir: {}
      containers:
      - name: gcc
        image: gcc:latest
        command: ["/bin/bash", "-c"]
        args:
        - >-
            sleep 3600s;
        volumeMounts:
            - name: scratch
              mountPath: /scratch
        resources:
          limits:
            memory: 40Gi
            cpu: "32"
            ephemeral-storage: 10Gi
          requests:
            memory: 40Gi
            cpu: "32"
            ephemeral-storage: 10Gi
      restartPolicy: Never
```

Start the pod after you put in your username in the yaml file:

```
kubectl apply -f scienceapp.yaml
```

Once the pod is running you can interactively access it:

```
kubectl exec -it scienceapp-<username> -- /bin/bash
```

### Start by cloning the repository and download the data 

We change to the local scratch directory and download data. This might take some time as we have ~1.3GB to download.

```
cd /scratch
git clone https://gitlab.com/NERSC/N10-benchmarks/hmmsearch-benchmark.git
cd hmmsearch-benchmark
bash ./scripts/wget_data.sh
```

### Build the application code

Next we can run the build script that will download the source code and build it

```
cd /scratch/hmmsearch-benchmark
bach ./scripts/build.sh
```

### Run the application

Follow the steps below to run the code:

```
export HMM_BENCH=/scratch/hmmsearch-benchmark
export HMM_DATA=$HMM_BENCH/data
export HMM_SEARCH=$HMM_BENCH/hmmer-3.3.2/src/hpc_hmmsearch
export NCPU=16

nohup $HMM_SEARCH --cpu ${NCPU} -o out.txt --noali data/Pfam-A.hmm data/uniprot_sprot.fasta &
```

The code will take some time to run. You can use the "top" command to see how its doing and also look at the out.txt file for output.

You can delete the pod once you have seen the output:
```
kubectl delete -f scienceapp.yaml
```

## AI training using PyTorch example

You can use the yaml files from the repo as is but please change the username to something unique as we are all sharing the same namespace. Please download the yaml files using the download raw file button 
<img src="https://github.com/user-attachments/assets/4033a577-e8d3-4909-9773-30ab39cafef3" width="256" style="vertical-align:middle"/>
and edit 

We start by running the training example:

```
kubectl apply -f pytorch-training.yaml
```
Check on status:

```
kubectl get pods
```

When the job is in run state you can check the logs for output:

```
kubectl logs gp3-username
```

Once you are done exploring, please delete the pod:

```
kubectl delete -f pytorch-training.yaml
```

## Text generation inference example

Start up the inference pod:

```
kubectl apply -f tgi-inference.yaml
```

Once the pod is running, get interactive access to the pod:

```
kubectl exec -it tgi-username -- /bin/bash
```

Once in the pod, start a python3 interpreter and then run:

```
from huggingface_hub import InferenceClient
client = InferenceClient(model="http://0.0.0.0:80")
for token in client.text_generation("Who made cat videos?", max_new_tokens=24, stream=True): print (token)
```

**Please make sure you did not leave any running pods. Jobs and associated completed pods are OK.**

