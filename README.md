
## Single-cell Graph Cuts Optimization (scGCO)

### Overview

**scGCO** is a method to identify genes demonstrating position-dependent differential expression patterns, also known as spatially viable genes, using the powerful graph cuts algorithm. ScGCO can analyze spatial transcriptomics data generated by diverse technologies, including but not limited to single-cell RNA-sequencing, or *in situ* FISH based methods.

### Repo Contents

This repository contains source codes of scGCO, and tutorials on running the program.

### Installation Guide

The primary implementation is as a Python 3 package, and can be installed from the command line by


```python
 pip install scgco
```


**scGCO** has been tested on Ubuntu Linux (18.04.1), Mac OS X (10.14.1) and Windows(Windows 7 Professional).

### License
MIT Licence, see LICENSE file.

###  Authors
See AUTHORS file.

### Contact
For bugs,feedback or help you can contact Peng Wang <wangpeng@picb.ac.cn>.

## Example usage of scGCO

The following codes demonstrate the typical data analysis flow of scGCO. 

The tutorial has also been provided as a Jupyter Notebook in the **Tutorial** directory (scGCO_tutorial.ipynb)

The entire process should only take 1-2 minutes on a typical desktop computer with 8 cores.

### Input Format
The required matrix format is the ST data format, a matrix of counts where spot coordinates are row names and the gene names are column names. This matrix format (.TSV) is split by tab.

As an example, let’s analyze spatially variable gene expression in Mouse Olfactory Bulb using a data set published in [Ståhl et al 2016](http://science.sciencemag.org/content/353/6294/78). 


```python
import scGCO

# read spatial expression data
ff = 'notebooks/data/Rep11_MOB_count_matrix-1.tsv'
locs, data = scGCO.read_spatial_expression(ff)
# remove genes expressed in less than 10 cells
data = data.loc[:,(data != 0).astype(int).sum(axis=0) >= 10]
# normalize expression and use 1000 genes to test the algorithm
data_norm = scGCO.normalize_count_cellranger(data)
data_norm = data_norm.iloc[:,0:1000]
import time
# estimate number of segments
factor_df, size_factor = scGCO.estimate_smooth_factor(locs, data_norm)
start_ts = time.time()
# run the main algorithm to identify spatially expressed genes
# this should take less than a minute 
result_df = scGCO.identify_spatial_genes(locs, data_norm, size_factor)
end_ts = time.time()
print('seconds to run: ', end_ts-start_ts)
```

    100%|██████████| 8/8 [00:05<00:00,  1.51it/s]
    100%|██████████| 8/8 [00:24<00:00,  8.44s/it]


    seconds to run:  25.059022665023804



```python
import warnings
warnings.filterwarnings('ignore')
# select spatially expressed gene with fdr < 0.01
fdr01 = result_df[result_df.fdr < 0.01].sort_values(by=['fdr'])
# visualize top genes
scGCO.visualize_spatial_genes(fdr01.iloc[4:6,], locs, data_norm)

# save top genes to pdf
scGCO.multipage_pdf_visualize_spatial_genes(fdr01.iloc[0:10,], locs, data_norm, 'notebooks/figures/top10_genes.pdf')
```


![png](notebooks/figures/top_genes.png)



```python
# Do PCA + t-SNE to visualize the clustering patterns of identified genes
# Though only 1000 genes are used, the pattern should resemble Fig. 2b in the manuscript
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

tsne_proj = scGCO.spatial_pca_tsne(data_norm, fdr01.index)
zz = scGCO.visualize_tsne_density(tsne_proj)
```


![png](notebooks/figures/density.png)



```python
# You can also analyze one gene of interest

geneID='Nrgn' # Lets use Nrgn as an example
unary_scale_factor = 100 # scale factor for unary energy, default value works well

# set smooth factor to 20; 
# use bigger smooth_factor to get less segments
# use small smooth_factor to get more segments
smooth_factor=20 

ff = 'notebooks/data/Rep11_MOB_count_matrix-1.tsv' 
# read in spatial gene expression data
locs, data = scGCO.read_spatial_expression(ff)

# normalize gene expression
data_norm = scGCO.normalize_count_cellranger(data)

# select Nrgn's expression
exp =  data_norm.loc[:,geneID]

# log transform
exp=(scGCO.log1p(exp)).values

# create graph representation of spatial coordinates of cells
cellGraph = scGCO.create_graph_with_weight(locs, exp)

# do graph cut
newLabels, gmm = scGCO.cut_graph_general(cellGraph, exp, unary_scale_factor, smooth_factor)
# calculate p values
p, node, com = scGCO.compute_p_CSR(newLabels, gmm, exp, cellGraph)

# Visualize graph cut results
scGCO.plot_voronoi_boundary(geneID, locs, exp,  newLabels, min(p)) 

# save the graph cut results to pdf
scGCO.pdf_voronoi_boundary(geneID, locs, exp, newLabels, min(p),  'Nrgn.pdf')

```


![png](notebooks/figures/Nrgn.png)



```python

```
## Reproducibility

Two Jupyter Notebooks are provided in the **Analysis** directory to reproduce figures of the main text. 

Both should take less than 1 minute to finish.

* gen_FIG2_A_B_D.ipynb: this notebook will reproduce Fig.2 A, B and D.

* gen_FIG2_C_E_F.ipynb: this notebook will reproduce Fig.2 C, E and F.

Several Jupyter Notebooks are provided in the Simulation directory to reproduce the running time simulation results reported in the main text.




### Simulating large data sets

The following three scripts simulated greater number of cells and will take substantially more time to finish.

The 1M simulation takes about 3 hours using a typical 8 cores computer.

* scGCO_simulate_100k.ipynb: benchmark the running time of scGCO using simulated data with 100K cells.
* scGCO_simulate_500k.ipynb: benchmark the running time of scGCO using simulated data with 500K cells.
* scGCO_simulate_1M.ipynb: benchmark the running time of scGCO using simulated data with 1M cells.
* scGCO_simulate_script.ipynb: benchmark the running time of scGCO using simulated data with cell numbers upto 10K. 

This script should take less than 2 minutes to finish on a typical 8 cores computer.
