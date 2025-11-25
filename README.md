# expression_with_python

basic setup
```
module load python

python -m venv ~/envs/scanpy
source ~/envs/scanpy/bin/activate

python -m ensurepip --upgrade
pip install --upgrade pip

pip install fsspec
```
checking file structure of h5ad files
( I am still trying this inside python)
```stream_test.py
import fsspec, scanpy as sc

print(fsspec.__version__)
print(sc.__version__)

url = "https://huggingface.co/datasets/arcinstitute/SE-167M-Human/resolve/main/19k_human_filtered_scbasecount/SRX10000417.h5ad"

with fsspec.open(url, "rb") as f:
    adata = sc.read_h5ad(f)

print(adata)
```
Inspecting whats inside

```
#aData
print(adata)
#gene level metadata
print(adata.var.head())
#cell level metedata
print(adata.obs.head())

```
In adata.var,
On the same row, n_cells_by_counts tells you how many cells expressed the gene, while total_counts tells you how many transcripts of that gene were captured across all cells. Together, they let you distinguish between ubiquitous low‑level expression and rare high‑level expression.

Now, because huggingface has the Arc dataset in a streamable and easy to access format, I am going to work with hugging face

It has different h5ad files for different SRX numbers/experiments that represent different cell types

First I am trying to count the number of SRX files and come up with a list

Install rust (for hugging face hub installation)
THIS IS OUTSIDE PYTHON ENVIRONMENT
```
curl https://sh.rustup.rs -sSf | sh
```
then re start the shell
load python 
and install hugging face
```
module load python
pip install --user huggingface_hub
```
verify by
```
python -c "import huggingface_hub; print(huggingface_hub.__version__)"
```
Then I had to register for huggingface through website
and created a token to access data

then you can test your access code inside python
```
python
from huggingface_hub import HfApi, list_repo_files

# Paste your token here
token = "hf_your_token_here"

# Test authentication
api = HfApi()
print(api.whoami(token=token))  # should show your username/orgs
```
load python and count files
```
module load python
python

from huggingface_hub import list_repo_files

token = "hf_your_token_here"

repo_id = "arcinstitute/SE-167M-Human"

# Important: set repo_type="dataset"
files = list_repo_files(repo_id, repo_type="dataset", token=token)

h5ad_files = [f for f in files if f.endswith(".h5ad")]

print("Number of .h5ad files:", len(h5ad_files))
print("First 10 files:", h5ad_files[:10])
```
# opening metadata files for cell types (inside python)

install packages that are not already installed like following
```
pip install --user scanpy
pip install pyarrow
```
and load arrow and then python
```
module load gcc arrow

module load python

python -m venv ~/envs/scanpy
source ~/envs/scanpy/bin/activate
```
Then open python and set environment
```
import os
import pandas as pd
import scanpy as sc
import pyarrow.dataset as ds
import gcsfs

# initialize GCS file system for reading data from GCS
fs = gcsfs.GCSFileSystem()

# GCS bucket path
gcs_base_path = "gs://arc-scbasecount/2025-02-25/"

# STARsolo feature type
feature_type = "GeneFull_Ex50pAS"

# helper function to list files 
def get_file_table(gcs_base_path: str, target: str=None, endswith: str=None):
    files = fs.glob("/".join([gcs_base_path.rstrip("/"), "**"]))
    if target:
        files = [f for f in files if os.path.basename(f) == target]
    else:
        files = [f for f in files if f.endswith(endswith)]
    file_list = []
    for f in files:
        file_list.append(f.split("/")[-2:-1] + [f])
    return pd.DataFrame(file_list, columns=["organism", "file_path"])

# set the path to the metadata files
gcs_path = "/".join([gcs_base_path.rstrip("/"), "metadata", feature_type])
gcs_path

```
Then you can list per-sample metadata filess by
```
# list files
sample_pq_files = get_file_table(gcs_path, "sample_metadata.parquet")
print(sample_pq_files.shape)
sample_pq_files.head()
```
List per-obs metadata files
```
# list files
obs_pq_files = get_file_table(gcs_path, "obs_metadata.parquet")
print(obs_pq_files.shape)
obs_pq_files.head()
```
h5ad files
```
# set the path
gcs_path = "/".join([gcs_base_path.rstrip("/"), "h5ad", feature_type])
gcs_path

# list files
h5ad_files = get_file_table(gcs_path, endswith=".h5ad")
print(h5ad_files.shape)
h5ad_files.head()
```
# ** Just human samples
```
# get the per-sample metadata file path
infile = sample_pq_files[sample_pq_files["organism"] == "Homo_sapiens"]["file_path"].values[0]
infile

# load the metadata
sample_metadata = ds.dataset(infile, filesystem=fs, format="parquet").to_table().to_pandas()
print(sample_metadata.shape)
sample_metadata.head()
```
now filter rows without proper cell line names
```
import pandas as pd

# Define the unwanted values
bad_values = ["not applicable", "not specified", "not_applicable"]

# Normalize to lowercase for consistent matching
sample_metadata["cell_line_norm"] = sample_metadata["cell_line"].str.strip().str.lower()

# Keep only rows that are NOT in bad_values
filtered_metadata = sample_metadata[~sample_metadata["cell_line_norm"].isin(bad_values)]

# Drop rows where cell_line is missing/null
filtered_metadata = filtered_metadata.dropna(subset=["cell_line"])

# Check result
print(filtered_metadata.head())
```




