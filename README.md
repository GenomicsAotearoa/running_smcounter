## SMCounter v2 notes

This README provides some notes on getting the qiaseq-dna analysis pipeline up and running via singularity:

https://github.com/qiaseq/qiaseq-dna

Figure 1 of the following publication provides an overview of the steps in the smcounter v2 workflow:

https://academic.oup.com/bioinformatics/article/35/8/1299/5091498

#### Issues/notes
- `qiaseq-dna` is the github repo that describes the use of the qiaseq pipeline - includes instructions  on how to use docker to run the whole pipeline, including smcounter: v1 or v2.
- when running singularity, need to be careful about existing setup files on the system, particularly files that alter the R environment.  The qiaseq docker container is running a very old R version: 3.2.3, so you need to make sure that it doesn't try to reference existing R packages on the underlying host system (e.g., via library directories specified in `.Renviron` files etc)

#### What is needed to run the qiaseq pipeline
- paired end fastq data
- bed file for the specific gene panel being used
- primer file for the specific gene panel being used
- edit the file: `run_sm_counter_v2.params.txt` (near the end) to specifiy the above details.

#### Setup (as per instructions at https://github.com/qiaseq/qiaseq-dna):
- google-cloud-sdk (to get `gsutils`); https://cloud.google.com/sdk/docs/quickstart-linux
    - `curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-260.0.0-linux-x86_64.tar.gz`
    - creates `google-cloud-sdk` folder containing (among many other things `bin/gsutil`)
- `gsutil` to download annotation data (https://github.com/qiaseq/qiaseq-dna):
   ```
   ### Get data dependencies, this will create a directory named data in your current folder
   gsutil -m cp -r gs://qiaseq-dna/data ./
   ```
- Example data (I saved this to the directory `example`)
   ```bash
   ### cd to your_fav_dir and get example fastqs, roi and primer files
   wget https://storage.googleapis.com/qiaseq-dna/example/
        NEB_S2_L001_R1_001.fastq.gz \
        https://storage.googleapis.com/qiaseq-dna/example/
        NEB_S2_L001_R2_001.fastq.gz \
        https://storage.googleapis.com/qiaseq-dna/example/
        DHS-101Z.primers.txt \
        https://storage.googleapis.com/qiaseq-dna/example/
        DHS-101Z.roi.bed ./
   ```
- `qiaseq-dna` (https://github.com/qiaseq/qiaseq-dna)
    - the following files from the github repo are needed to run the pipeline:
      - `run_qiaseq_dna.py`
      - `run_sm_counter_v2.params.txt`
    - the instructions suggest cloning the github repo inside once the container is running, but because the dockerfile sets the target directory up as being owned by root, this won't work under singularity.
    - to get around this, I simply cloned the repo to a directory on the host system, and then mounted that when the container was loaded.
- Build container: `singularity pull docker://qiaseq/qiaseq-dna`
    - this will produce the container: `qiaseq-dna_latest.sif `
    - __NOTE:__ I had a "no space left on device" error when trying to rebuild.  Solved by defining a `tmp` directory:
    ```bash
    tmpdir=[SOME TEMPORARY DIRECTORY]
    export SINGULARITY_LOCALCACHEDIR=${tmpdir}
    export SINGULARITY_TMPDIR=${tmpdir}
    export SINGULARITY_CACHEDIR=${tmpdir}
    ```

#### To run the container:
```
singularity run qiaseq-dna_latest.sif
```
The directory `/src/qgen/` contains the subdirectories `bin` and `code`
- `bin` contains the software needed to run the pipeline
- `code` is empty, and is designed to hold the cloned `qiaseq-dna` github repo (but it is root-owned: see above)

```bash
DIRPATH=[WHERE YOU ARE WORKING]
singularity run --writable-tmpfs \
  -B $DIRPATH/data:/srv/qgen/data \
  -B $DIRPATH/example:/srv/qgen/example \
  -B $DIRPATH/code2:/srv/qgen/code2  \
  qiaseq-dna_latest.sif
```

#### Directory notes:
- `/srv/qgen/data` - annotation data downloaded via `gsutil` (see above)
- `/srv/qgen/code2` - cloned `qiaseq-dna` github repo (see above)
- `/srv/qgen/example` - location of fastq files, and bed and primer files for gene panel being used.

#### Inside singularity
 - Change to the directory where the fastq, bed and primer files are: `cd /srv/qgen/example`
 - Make a local copy of the run file: `cp /srv/qgen/code/qiaseq-dna/run_sm_counter_v2.params.txt ./`
 - Add details of the fastq, bed and primer files to the end of `run_sm_counter_v2.params.txt`
 - Run the pipeline:
   ```bash
   python /srv/qgen/code2/qiaseq-dna/run_qiaseq_dna.py run_sm_counter_v2.params.txt v2 \
      single crc_47 > run.log 2>&1 &  
   ```

#### Qiagen primer and bed files:
- Colorectal cancer: panel code = 002Z
    - `DHS-002Z.roi.bed.txt`
    - `DHS-002Z.primer3.txt`
- Breast cancer: panel code = 001Z
    - `DHS-001Z.primer3.txt`
    - `DHS-001Z.roi.bed`
