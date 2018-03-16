# Survival Convolutional Networks
This page contains software and data resources related to the paper

>[Mobadersany, *et al.* "Predicting cancer outcomes from histology and
genomics using convolutional networks" *PNAS* published online March 12, 2018 ahead of print](http://www.pnas.org/content/early/2018/03/09/1717139115).

We provide scripts for formatting and analyzing data, a portable Docker container that encapsulates an executable software, documentation, and data used to generate the results shown in the paper.

## 3/12/2018 Update
We are working to document and organize the software and scripts. We anticipate that everything will posted here by Friday 3/16. We apologize for the delay.

## Data
The results in this paper were generated using whole-slide .svs images of paraffin embedded sections and clinica outcomes data from The Cancer Genome Atlas. These images are publically available and hosted at the [NCI Genomic Data Commons](https://gdc.cancer.gov/) (GDC) Legacy Archive. A full list of the whole-slide image files used in the paper is available in /data/rois.txt. Note that the whole-slide image files and even the extracted regions-of-interest (ROIs) are quite large and so these are not hosted here.

### Downloading the data
GDC does not currently enable direct querying of the TCGA diagnostic images for a specific project. To generate a list of the files to download, you have to first generate a manifest of all whole-slide images in TCGA (both frozen and diagnostic), filter the frozen section images in this list, and then match the identifiers against the sample identifiers (TCGA-##-####) for the project(s) of interest.

The manifest for all TCGA whole-slide images can be generated using the [GDC Legacy Archive query](https://portal.gdc.cancer.gov/legacy-archive/search/f?filters=%7B%22op%22:%22and%22,%22content%22:%5B%7B%22op%22:%22in%22,%22content%22:%7B%22field%22:%22files.data_format%22,%22value%22:%5B%22SVS%22%5D%7D%7D%5D%7D).

Rows containing diagnostic image files can be identified using the Linux command line
```
cut -d$'\t' -f 2 gdc_manifest.txt | grep -E '\.*-DX[^-]\w*.'
```

After matching the slide filenames against the sample IDs from the clinical data for the project(s) of interest, the relevant filenames can be used with the [GDC Data Transfer Tool](https://gdc.cancer.gov/access-data/gdc-data-transfer-tool) or the [GDC API](https://gdc.cancer.gov/developers/gdc-application-programming-interface-api).

### Extracting regions of interest
Regions of interest can be extracted using the python script generate_rois.py. This script consumes a tab-delimited text file describing the whole-slide image files, ROI coordinates, desired size and magnification for extracted ROIs, and then generates a collection of ROI .png images. These images are transformed into a binary for model training and testing by the software described below.

*Note: region extraction depends on the [OpenSlide](http://openslide.org/) library. This library can be installed locally, and is also available in the Docker image described below.*

# Executables for training and testing models

The Docker container provides executables for training SCNN/GSCNN models and for evaluating the accuracy of these models. Both of these executables consume:
* ROI .png files
* A .csv file describing the patient IDs, event times, censoring (0 for uncensored, 1 for right-censored), patient indexes (integer indexes assigned to each patient), and any genomic variables. **_Note that the columns for genomic data should be organized in a way that the first coloumns are allocated for those which will NOT get normalized and the latter ones allocated for those that have to get normnalized._** The default normalization method in our model is Z score normalization. An example file is provided in this repository.

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
usage: model_train.py [-h] [-m M] [-p P] [-f F] [-i I] [-r R] [-t T] [-d D]
                      [--lr LR] [--me ME] [--kp KP] [--bs BS] [--ic IC]
                      [--ngf NGF] [--gn GN] [--ns NS] [--nm NM]

Arguments for Training the SCNN/GSCNN model

optional arguments:
  -h, --help  show this help message and exit
  -m M        image_only or image_genome; image_only for SCNN and image_genome
              for GSCNN. The default is image_only.
  -p P        Path to the Train/Test patient list. The default path is ./inputs/ImageSplits.txt
  -f F        Path to the features data. The default path is ./inputs/all_dataset.csv
  -i I        Path to the folder containing ROI .pngs. The default path is ../images/train
  -r R        Path where the training outputs should be generated. The default path is ./train_results
  -t T        Path where training meta files should be generated. The default path is ./checkpoints
  -d D        Path where binary files for training should be generated (these can be quite large and
              should be cleaned). The default path is ./tmp
  --lr LR     Initial learning rate. Default value = 0.001.
  --me ME     Max number of epochs. Default value = 100.
  --kp KP     Keeping probability for Test time dropout. Default value = 0.95.
  --bs BS     batch size. Default value = 14.
  --ic IC     indexes column in the number in the ./inputs/all_dataset.csv
              file, starting from 0. Default value = 1.
  --ngf NGF   Number of genomic features. The default is 2, considering the
              case we have just "codeletion" and "idh mutation" as genomic
              features.
  --gn GN     Z Normalization of the geomic values; y for yes and n for no.
              Default value = n.
  --ns NS     Starting column number of Normalization for the genomic values
              we want to do the Z normalization. This is the column number in
              the -f argument file, starting from 0 index. Default value = 6.
              Note: The columns we want to normalize should get assigned
              as the last columns of the all_data.csv file.
  --nm NM     Number of the latest models we want to save for the test time.
              Default value = 5.
```
The outputs generetaed by model_train.py are:

* Binary conversion of .png ROI files
* ###_all_features.csv saved in ./train_results directory by default. ### is the date/time when the executable was run. This file contains four columns: TCGA ID (TCGA IDs of each patient), indexes (allocated index for each patient), censored (censoring status of each patient: 0 for NOT censored, 1 for censored), Survival months (survival time of each patient in months).
* ###_training_loss.txt saved in ./train_results directory by default. ### is the date/time when the executable was run. This file contains one column which represents the training loss value at each epoch.
* Trained models saved at TensorFlow checkpoints.

**Example**
```
python model_train.py -m image_genome --ngf 80 --gn y --ns 6
```
This command will train the GSCNN model with 80 genomic features where normalization is applied starting from column 6.

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
                     [-d D] [-f F] [--ic IC] [--ngf NGF] [--gn GN] [--ns NS]
                     [--nm NM]

Arguments for prediction using an SCNN/GSCNN model

optional arguments:
  -h, --help  show this help message and exit
  -m M        image_only or image_genome; image_only for SCNN and image_genome
              for GSCNN. The default is image_only
  --kp KP     Keeping probability for Test time dropout. Default value = 1.
  --bs BS     batch size. Default value = 14.
  -i I        Path to the folder containing ROI .pngs. The default path is ../images/test
  -r R        Path where testing results will be generated. The default path is ./test_results
  -t T        Path to the training meta files. The default is ./checkpoints
  -d D        Path where the temporary binary files for testing will be generated. The default path is ./tmp
  -f F        Path to the features data. The default path is ./inputs/all_dataset.csv
  --ic IC     indexes column in the ./inputs/all_dataset.csv file, starting
              from 0. Default value = 1
  --ngf NGF   Number of genomic features. Default value = 2.
  --gn GN     Z-score normalization of the genomic features; y for yes and n for no.
              Default value = n.
  --ns NS     Starting column number of Normalization for the genomic values
              we want to do the Z normalization. This is the column number in
              the ./inputs/all_dataset.csv file, starting from 0. Default
              value = 6; Note: The columns we want to normalize should get assigned
              as the last columns of the all_data.csv file
  --nm NM     number epochs for model averaging. Default value = 5.
``` 
The outputs generetaed by model_test.py are:
 
* Input binary files saved in ./tmp folder by default
* ###_testset_cindex.txt saved in the folder specified by the -r argument. ### is the date/time when the executable was run. This file contains the final C-Index of the model during testing. As described in the paper, this C-Index represents the accuracy of model-averaged risks (the last 5 models by default that are saved during training). [See the *Methods* section for more details](http://www.pnas.org/content/early/2018/03/09/1717139115).
* ###_testset_patients_risks.csv saved in ./train_results directory by default. ### is the date/time when the executable was run. This file contains four columns that represent: indexes (allocated index of each testset patient), censored (censoring status for each testset patient: 0 for NOT censored, 1 for censored), survivals (survival time for each testset patient in months), risks (predicted risk values by the model for each testset patient)

**Example**
```
python model_test.py -m image_genome --ngf 80 --gn y --ns 6
```
This command will test the GSCNN model with 80 genomic features where normalization is applied starting from column 6.

*Note that by default this model is using the last 5 models saved during training in order to do the prediction for test patients.*
