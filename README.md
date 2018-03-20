# 3/20/2018 Update
The Docker image is available below. We are still working to organize and document the data analysis scripts. We apologize for the delay.

# Survival Convolutional Networks
This page contains software and data resources related to the paper

>[Mobadersany, *et al.* "Predicting cancer outcomes from histology and
genomics using convolutional networks" *PNAS* published online March 12, 2018 ahead of print](http://www.pnas.org/content/early/2018/03/09/1717139115).

We provide scripts for formatting and analyzing data, a portable Docker container that encapsulates an executable software, documentation, and data used to generate the results shown in the paper.

# Docker container

A Docker container that encapsulates executables and data is posted at [DockerHub](https://hub.docker.com/r/cancerdatascience/scnn/)

Brief directions for deploying this Docker are provided below. Consult the [Docker Tutorial](https://docs.docker.com/get-started/) for additional guidance.

1. Pull the docker to your system and start the container.

``
$docker pull cancerdatascience/scnn:1.0
``

2. Confirm that the docker image is downloaded. The image is > 10 GB due to the size of the data contained in the image. 

``
$docker images
REPOSITORY                     TAG                            IMAGE ID            CREATED             SIZE
cancerdatascience/scnn         1.0                            21d851ebf629        20 minutes ago      13.2GB
``

3. Switch to the docker container and run the code on CPU or GPU

3-1. CPU version

``
$docker run -it cancerdatascience/scnn:1.0 /bin/bash
root@97d439b58033:/# cd /root/scnn
root@97d439b58033:/# python model_train.py
``

3-2. GPU version (4 GPUs - see note below)

``
$docker run --device=/dev/nvidiactl --device=/dev/nvidia-uvm --device=/dev/nvidia0:/dev/nvidia0 --device=/dev/nvidia1:/dev/nvidia1 --device=/dev/nvidia2:/dev/nvidia2 --device=/dev/nvidia3:/dev/nvidia3 -i -t cancerdatascience/scnn:1.0 /bin/bash
root@97d439b58033:/# cd /root/scnn
root@97d439b58033:/# python model_train.py
``

*Note: this docker is built on CUDA 8.0 with CUDNN 5.1 and Nvidia driver 367.57. This code was developed on a system with 4 NVIDIA K80 GPUs. The memory limitations of GPU systems vary widely and running this Docker in GPU with inadequate resources may produce memory errors.*

# Executables for training and testing models

The Docker container provides executables for training SCNN/GSCNN models and for evaluating the accuracy of these models. Both of these executables consume:
* ROI .png files
* A .csv file describing the patient IDs, event times, censoring (0 for uncensored or event observed, 1 for right-censored), patient indexes (integer indexes used to refer to patients inside the binary files that are used in training and testing), and any genomic variables. An example file is provided in this repository.

*Note: Executables may not produce identical models or statistics for different runs. We have taken steps to reduce variability wherever possible, but the issues of non-associativity in distributed and GPU computing produce variations that stack as models are trained. Graph-level seeding and operation-level seeding issues were addressed using TensorFlow functions to eliminate these as a source of randomness.*

## Training

The command
```
python model_train.py
```
will train a model using the default hyperparameters and data locations described below.

The input parameters and their default values can be listed using
```
python model_train.py -h
```
that generates output
```
usage: model_train.py [-h] [-m M] [-f F] [-i I] [-r R] [-t T] [-d D] [--lr LR]
                      [--me ME] [--kp KP] [--bs BS] [--ic IC] [--ngf NGF]
                      [--nm NM]

Arguments for Training the SCNN/GSCNN model

optional arguments:
  -h, --help  show this help message and exit
  -m M        SCNN or GSCNN; SCNN for the case we just use histology images
              and GSCNN for the case we integrate the histology and genomic
              features. Default value = SCNN.
  -f F        Path to the file containing patient IDs, patient indexes, 
              clinical outcomes, and genomic features. The default path and 
              filename is ./inputs/all_dataset.csv
  -i I        Path to the Training ROIs. The default is ../images/train
  -r R        Path to the Training results (training loss and other outputs).
              The default path is ./train_results
  -t T        Path to save the trained models and their
              weights/biases/parameters values. The default is ./checkpoints
  -d D        Path to the temporary binary files for Training. The default path 
              is ./tmp
  --lr LR     Initial learning rate. Default value = 0.001.
  --me ME     Max number of epochs. Default value = 100.
  --kp KP     Keeping probability for training weight dropout. Default value = 0.95.
  --bs BS     Batch size. Default value = 14.
  --ic IC     Column containing patient indices in the -f input (0-indexed). 
              Default value = 1.
  --ngf NGF   Number of genomic features in the -f input. Default value = 2.
  --nm NM     Number models to save for test time model averaging. 
              Default value = 5.
```
The outputs generetaed by model_train.py are:

* Binary conversion of .png ROI files for train-set
* ###_all_features.csv saved in -r path (./train_results by default). ### is the date/time when the training executable was run. This file contains four columns: TCGA ID (TCGA IDs of each patient), indexes (integer index for each patient), censored (censoring status of each patient: 0 for uncensored or event observed, 1 for right-censored), event or last followup times. This file contains inputs as converted in the binary file to confirm data alignment and correct conversion and scaling of any genomic features.
* ###_training_loss.txt saved in -r path (./train_results directory by default). ### is the date/time when the executable was run. This file contains one column which represents the training loss value at each epoch. These loss values are the summation of negative partial log likelihood and weight decay penalty at each epoch. This is used to evaluate model convergence.
* Trained models saved at TensorFlow checkpoints. These model files are used during testing.

**Example**
```
python model_train.py -m GSCNN --ngf 80
```
This command will train the GSCNN model with 80 genomic features.

## Testing

The command
```
python model_test.py
```
will generate risk values for testing patients using a trained model. The testing input parameters and their default values for the model_test.py can be listed using the command
```
python model_test.py -h
```
that generates output
```
usage: model_test.py [-h] [-m M] [--kp KP] [--bs BS] [-i I] [-r R] [-t T]
                     [-d D] [-f F] [--ic IC] [--ngf NGF] [--nm NM]

Arguments for Testing the SCNN/GSCNN model

optional arguments:
  -h, --help  show this help message and exit
  -m M        SCNN or GSCNN; SCNN for the case we just use the clinical
              outcomes and GSCNN for the case we integrate the clinical
              outcomes with genomic features. Default value = SCNN.
  --kp KP     Keeping probability for test weight dropout. Default value = 1.
  --bs BS     Batch size. Default value = 14.
  -i I        Path to the folder containing ROI .pngs for testing. 
              The default path is ../images/test
  -r R        Path where testing results will be generated (final test C-Index and 
              patient risk values). The default path is ./test_results
  -t T        Path to folder containing the trained models and their
              weights/biases/parameters values. The default path is ./checkpoints
  -d D        Path where the binary files for testing will be generated. 
              The default path is ./tmp
  -f F        Path to the input .csv file. The default path and filename is
              ./inputs/all_dataset.csv
  --ic IC     Column containing patient indices in the -f input (0-indexed). 
              Default value = 1.
  --ngf NGF   Number of genomic features in the -f input file. Default value = 2.
  --nm NM     number of models used for model averaging during testing. 
              Default value = 5.
``` 
The outputs generetaed by model_test.py are:
 
* Binary conversion of .png ROI files for test-set
* ###_testset_cindex.txt saved in the folder specified by the -r argument. ### is the date/time when the executable was run. This file contains the final C-Index of the model during testing. As described in the paper, this C-Index represents the accuracy of model-averaged risks (the last 5 models by default that are saved during training). [See the *Methods* section for more details](http://www.pnas.org/content/early/2018/03/09/1717139115).
* ###_testset_patients_risks.csv saved in ./train_results directory by default. ### is the date/time when the executable was run. This file contains four columns that represent: indexes (allocated index of each testset patient), censored (censoring status for each testset patient: 0 for uncensored or event observed, 1 for right-censored), survivals (survival time or time-to-event for each testset patient), risks (predicted risk values by the SCNN/GSCNN model for each testset patient). This file contains the predicted patient risks used to calculate the final c-index. It also contains the inputs as converted in the binary file to confirm data alignment and correct conversion and scaling of any genomic features.

**Example**
```
python model_test.py -m GSCNN --ngf 80
```
This command will test the GSCNN model with 80 genomic features.

# Data
The results in this paper were generated using whole-slide .svs images of paraffin embedded sections and clinica outcomes data from The Cancer Genome Atlas. These images are publically available and hosted at the [NCI Genomic Data Commons](https://gdc.cancer.gov/) (GDC) Legacy Archive. A full list of the whole-slide image files used in the paper is available in /data/rois.txt.

## Downloading the data
GDC does not currently enable direct querying of the TCGA diagnostic images for a specific project. To generate a list of the files to download, you have to first generate a manifest of all whole-slide images in TCGA (both frozen and diagnostic), filter the frozen section images in this list, and then match the identifiers against the sample identifiers (TCGA-##-####) for the project(s) of interest.

The manifest for all TCGA whole-slide images can be generated using the [GDC Legacy Archive query](https://portal.gdc.cancer.gov/legacy-archive/search/f?filters=%7B%22op%22:%22and%22,%22content%22:%5B%7B%22op%22:%22in%22,%22content%22:%7B%22field%22:%22files.data_format%22,%22value%22:%5B%22SVS%22%5D%7D%7D%5D%7D).

Rows containing diagnostic image files can be identified using the Linux command line
```
cut -d$'\t' -f 2 gdc_manifest.txt | grep -E '\.*-DX[^-]\w*.'
```

After matching the slide filenames against the sample IDs from the clinical data for the project(s) of interest, the relevant filenames can be used with the [GDC Data Transfer Tool](https://gdc.cancer.gov/access-data/gdc-data-transfer-tool) or the [GDC API](https://gdc.cancer.gov/developers/gdc-application-programming-interface-api).

### Extracting regions of interest
Regions of interest can be extracted using the python script generate_rois.py. This script consumes a tab-delimited text file describing the whole-slide image files, ROI coordinates, desired size and magnification for extracted ROIs, and then generates a collection of ROI .png images. These images are transformed into a binary for model training and testing by the software described below.

*Note: region extraction depends on the [OpenSlide](http://openslide.org/) library.*
