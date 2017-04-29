# DiiD-Predictor

## When deploying

* Install the virtualenv requirements
* Upload the existing ImageNet weights

## Setup

On a Google Cloud compute instance:

Install the GPU (instructions based on those from (Google Cloud docs)[https://cloud.google.com/compute/docs/gpus/add-gpus]):

```bash
echo "Checking for CUDA and installing."
if ! dpkg-query -W cuda; then
  # The 16.04 installer works with 16.10.
  curl -O http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_8.0.61-1_amd64.deb
  sudo dpkg -i ./cuda-repo-ubuntu1604_8.0.61-1_amd64.deb
  sudo apt-get update
  sudo apt-get install cuda -y
fi
```

Add the CudNN library:
From your main development computer (not Google Cloud), transfer the `deb` that you download from [here](https://developer.nvidia.com/cudnn) (make sure to download version 5.1).
```bash
scp -i ~/.ssh/google_compute_engine ~/Downloads/libcudnn6_6.0.20-1+cuda8.0_amd64.deb  andrew@35.185.45.28:/home/andrew/
```

Then from the compute machine, install it with `dpkg`:
```bash
sudo dkpg -i libcudnn6_6.0.20-1+cuda8.0_amd64.deb
```

Then do other project dependencies:

```bash
# sudo apt-get install git
git clone <link to this repository>

# sudo easy_install pip
sudo apt-get update
sudo apt-get install python-pip --fix-missing
pip install virtualenv

cd DiiD-Predictor
virtualenv venv -p python3
source venv/bin/activate
pip install -r requirements.txt
```

## Get images for a country

```bash
mkdir images
cd images/
gsutil -m cp -r gs://diid/Rwanda .
```

## Preprocess images to have the expected indexes

*Note*: The `rwanda_TL.csv` file comes will have to be uploaded securely (it's protected data).
Before doing this, create a `csv` directory in the `DiiD-Predictor` directory.
For uploading this file with `scp`, see related instructions for connecting over `ssh` (here)[https://cloud.google.com/compute/docs/instances/connecting-to-instance#standardssh].
Then, run the `copy_images.py` script.

```bash
python copy_images.py \
  images/Rwanda/ \
  csv/rwanda_TL.csv \
  --output-dir images/Rwanda_simple/
```

## Split into the test set

```bash
python util/get_test_indexes.py \
  images/Rwanda_simple/ 0.1 > indexes/Rwanda_test_indexes.txt
```

## Extract features from an arbitrary layer of a neural network model

```bash
python extract_features.py \
  images/Rwanda_simple/ \
  block5_pool \
  --output-dir features/rwanda_vgg16_block5_pool/
```

The `--flatten` flag is optional, and it flattens the feature array for each image.  Replace `"block5_conv3"` with the name of the layer you want to extract features for.  You can also set the number of images to process together with the `--batch-size` argument.  While this extracts features from an ImageNet VGG16 model, you can also provide another model with the `--model` option.

## Train the top layers of the neural network

To train the top layers:
```bash
python train_top.py \
  features/rwanda_vgg16_block5_pool \
  csv/rwanda_TL.csv \
  indexes/Rwanda_test_indexes.txt \
  --learning-rate 0.01 \
  --batch-size=100 \
  --epochs=6 \
  -v
```

Note that this steps depends on having run the previous steps to have the CSV file for Rwanda data, having the images moved into simple indexing form (single integer in the title), and having generated the test set image indexes.

## Get final layer features (can be used for predicting ed and wealth index)

We rely on the `extract_features` script once again:

```bash
python extract_features.py \
  features/rwanda_vgg16_block5_pool \ 
  predictions_dense \
  --model models/rwanda_vgg16_trained_top.h5
  --input-type=features
  --batch-size=16
  --output-dir=features/rwanda_vgg16_trained_top_predictions_dense/
```

## Retrain convolutional layers

Extract the features in block 4:

```bash
python extract_features.py \
  images/Rwanda_simple/ \
  block4_pool \
  --output-dir features/rwanda_vgg16_block4_pool/
```

Then retrain only the end of top of the net and the last convolutional block.
(At the time of writing this, this step wasn't yet implemented.)

```bash
python retrain_conv_block5.py \
  features/rwanda_vgg16_block4_pool \
  models/rwanda_vgg16_trained_top.h5 \
  csv/rwanda_TL.csv \
  indexes/Rwanda_test_indexes.txt \
  --batch-size=100 \
  --learning-rate=.0001 \
  --epochs=6 \
  -v
```

## Getting images that cause activation in later layers

```bash
python extract_features.py \
  features/rwanda_vgg16_block4_pool/ \
  block5_conv3 \
  --model models/rwanda_vgg16_retrained.h5
  --input-type=features
  --batch-size=16
  --output-dir=features/rwanda_vgg16_retrained_block5_conv3/
```

```bash
python get_activations.py \
  features/rwanda_vgg16_retrained_block5_conv3/
```
