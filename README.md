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
then python script
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



