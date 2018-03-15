# Survival Convolutional Networks
This page contains software and data resources related to the paper

>[Mobadersany, *et al.* "Predicting cancer outcomes from histology and
genomics using convolutional networks" *PNAS* published online March 12, 2018 ahead of print](http://www.pnas.org/content/early/2018/03/09/1717139115).

We provide scripts for formatting and analyzing data, a portable Docker container that encapsulates an executable software, documentation, and data used to generate the results shown in the paper.

## 3/12/2018 Update
We are working to document and organize the software and scripts. We anticipate that everything will posted here by Friday 3/16. We apologize for the delay.

## Data
The results in this paper were generated using whole-slide .svs images of paraffin embedded sections from The Cancer Genome Atlas. These images are publically available and hosted at the [NCI Genomic Data Commons](https://gdc.cancer.gov/) (GDC) Legacy Archive. A full list of the whole-slide image files used in the paper is available in /data/rois.txt. Note that the whole-slide image files and even the extracted regions-of-interest (ROIs) are quite large and so these are not hosted here.

### Downloading the data
GDC does not currently enable direct querying of the TCGA diagnostic images for a specific project. To generate a list of the files to download, you have to first generate a manifest of all whole-slide images in TCGA (both frozen and diagnostic), filter the frozen section images in this list, and then match the identifiers against the sample identifiers (TCGA-##-####) for the project(s) of interest.

The manifest for all TCGA whole-slide images can be generated using the [GDC Legacy Archive query](https://portal.gdc.cancer.gov/legacy-archive/search/f?filters=%7B%22op%22:%22and%22,%22content%22:%5B%7B%22op%22:%22in%22,%22content%22:%7B%22field%22:%22files.data_format%22,%22value%22:%5B%22SVS%22%5D%7D%7D%5D%7D):

Rows containing diagnostic images can be identified using the Linux command line to identify slides with names 
```
cut -d$'\t' -f 2 gdc_manifest.txt | grep -E '\.*-DX[^-]\w*.'
```

After matching against the sample IDs from the projects of interest, these filenames can be used with the GDC download tool or API.

### Extracting regions of interest
Regions of interest can be extracted using the python script generate_rois.py. This script consumes a tab-delimited text file describing the whole-slide image files, ROI coordinates, desired size and magnification for extracted ROIs, and then generates a collection of ROI .png images. These images are transformed into a binary for model training and testing by the software described below.
