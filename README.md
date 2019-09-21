<a href="https://pypi.org/project/minmlst"><img src="https://img.shields.io/pypi/v/minmlst"></a>
<a href="https://pypi.org/project/minmlst"><img src="https://img.shields.io/pypi/pyversions/minmlst"></a>
<a href="https://pypi.org/project/minmlst"><img src="https://img.shields.io/badge/platform-windows%20%7C%20linux-lightgrey"></a>
<a href="https://pypi.org/project/minmlst"><img src="https://img.shields.io/pypi/l/minmlst"></a>

# minMLST

minMLST is a machine-learning based methodology for identifying a minimal subset of genes
that preserves high discrimination among bacterial strains. It combines well known
machine-learning algorithms and approaches such as XGBoost, distance-based hierarchical
clustering, and SHAP. 

minMLST quantifies the importance level of each gene in an MLST scheme and allows the user 
to investigate the trade-off between minimizing the number of genes in the scheme vs preserving 
a high resolution among strain types.


## Installation

minMLST can be easily installed from [PyPI](https://pypi.org/project/minmlst):

<pre>
pip install minmlst
</pre>

## Usage
### 1. Quantifying gene importance

This function provides a ranking of gene importance according to selected measures: shap, weight, gain,
cover, total gain or total cover.
* **shap** - the mean magnitude of the SHAP values, i.e. the mean absolute value of the 
           SHAP values of a given gene ([Lundberg and Lee, 2017](http://papers.nips.cc/paper/7062-a-unified-approach-to-interpreting-model-predictions)).
* **weight** - the number of times a given gene is used to split the data across all splits.
* **gain** (or **total gain**) - the average (or total) gain is the average (or total) reduction of 
                             Multiclass Log Loss contributed by a given gene across all splits.
* **cover** (or **total cover**) - the average (or total) quantity of observations concerned by a given 
                               gene across all splits.
                               
As a pre-step, STs (strain types) with a single representative isolate are filtered from the dataset.
Next, an XGBoost model is trained with parameters `max_depth`, `learning_rate`, `stopping_method` and `stopping_rounds`
([more information about XGBoost parameters](https://xgboost.readthedocs.io/en/latest/python/python_api.html)); 
model's performance is evaluated by Multi-class log loss over a test set.
Finally, gene importance scores are measured for the trained model and provided as a DataFrame output.

A data sample of Legionella pneumophila cgMLST scheme can be downloaded from 
[here](https://github.com/shanicohen33/minMLST/blob/master/data/Legionella_pneumophila_cgMLST_sample.csv). 


```python
import pandas as pd
from os.path import join, sep
from IPython.display import display
from minmlst.tools import gene_importance, gene_reduction_analysis

path = join('C:', sep, 'Users', 'user_name', 'Desktop', 'minMLST_data')
data_file = 'Legionella_pneumophila_cgMLST_sample'

# load scheme
data = pd.read_csv(join(path, data_file + '.csv'), encoding="cp1252")
# define measures
measures = ['shap', 'total_gain', 'total_cover', 'weight', 'gain', 'cover']
# quantify gene importance according to selected measures using minmlst 
gi_results = gene_importance(data, measures)
# save results to csv
gi_results.to_csv(join(path, 'gene_importance_Legionella_pneumophila' + '.csv'), index=False)
# display results
display(gi_results)
```

<p align="center">
  <img width="811" src="/docs/gene_importance_results.png" />
</p>

### minmlst.tools.gene_importance
##### Parameters:
* `data` (DataFrame): 
    
    DataFrame in the shape of (**m**,**n**):
    **n-1** columns of genes, last column **n** must contain the ST (strain type).
    Each row **m** represents a profile of a single isolate.
    Data types should be integers only.
    Missing values should be represented as 0, no missing values are allowed for the ST (last column).
    
* `measures` (1-D array-like): 

    An array containing at least one of the following measures: 
    'shap', 'weight', 'gain', 'cover', 'total_gain' or 'total_cover'.
    
* `max_depth` (int, optional, default = 6): 

    Maximum tree depth for base learners. Must be greater equal to 0.
    
* `learning_rate` (float, optional, default = 0.3): 

    Boosting learning rate. Must be greater equal to 0.
    
* `stopping_method` (str, optional, default = 'num_boost_round'): 

    Stopping condition for the model's training process. Must be either 'num_boost_round' or 'early_stopping_rounds'.
    
* `stopping_rounds` (int, optional, default = 100): 

    Number of rounds for boosting or early stopping. Must be greater than 0.

##### Returns:
`importance_scores` (DataFrame):

Importance score per gene according to each of the input measures.
                    Higher scores are given to more important (informative) genes.
                 
<br>

### 2. Gene reduction analysis

This function analyzes how minimizing the number of genes in the MLST scheme impacts strain typing performance.
At each iteration, a reduced subset of most important genes is selected; and based on the allelic profile composed of these genes,
isolates are clustered into strain types (ST) using a distance-based hierarchical clustering (complete linkage).
The distance between every pair of isolates is measured by a normalized Hamming distance, which stands for
the proportion of those genes between the two allelic profiles which disagree.

To obtain a clustering structure (i.e. STs), we apply a threshold (or maximal distance between isolates of the same ST)
that equals to a certain percentile of distances distribution; This percentile (or percentiles) can be defined by
the user in the `percentiles` input parameter, or alternatively being selected by the **<em>find recommended threshold</em>** procedure
that searches in the search space of `percentiles_to_check` input parameter (see 2.3).

Typing performance is measure by the Adjusted Rand Index (ARI), which quantifies similarity between the induced
clustering structure (that is based on a subset of genes) and the original clustering structure (that is based on
all genes, and was given as an input) ([Hubert and Arabie, 1985](https://link.springer.com/article/10.1007/BF01908075)).
p-value for the ARI is calculated using a permutation test based on Monte Carlo simulation study ([Qannari et al., 2014](https://www.sciencedirect.com/science/article/abs/pii/S0950329313000852)).
The analysis results are provided as a DataFrame output and are also plotted by default.


#### 2.1 Using default parameters
```python
# define a measure
measure = 'shap'
# reduce the number of genes in the scheme based on the selected measure
# and compute the ARI results per subset of genes  
analysis_results = gene_reduction_analysis(data, gi_results, measure)
# save results to csv
analysis_results.to_csv(join(path, f'analysis_Legionella_pneumophila_{measure}' + '.csv'), index=False)
# display results
display(analysis_results)
```

<p align="center">
  <img width="811" src="/docs/analysis_results.png" />
</p>

### minmlst.tools.gene_reduction_analysis
##### Parameters:
* `data` (DataFrame): 
    
    DataFrame in the shape of (**m**,**n**):
    **n-1** columns of genes, last column **n** must contain the ST (strain type).
    Each row **m** represents a profile of a single isolate.
    Data types should be integers only.
    Missing values should be represented as 0, no missing values are allowed for the ST (last column).
    
* `gene_importance` (DataFrame): 

   Importance scores in the format returned by the **<em>gene_importance</em>** function.
    
* `measure` (str):

   A single measure according to which gene importance will be defined.
   It can be either 'shap', 'weight', 'gain', 'cover', 'total_gain' or 'total_cover'. 
   Note that the selected measure must be included in the `gene_importance` input.
    
* `reduction` (int or float, optional, default = 0.2): 
 
    The number (int) or percentage (0<float<1) of least important genes to be removed at each iteration.
    The first iteration includes all genes, the second iteration includes all informative genes (importance score > 0), 
    and the subsequent iterations include a reduced subset of genes according to the `reduction` parameter.
    
* `percentiles` (float or 1-D array-like of floats, optional, default = [0.5, 1]):

    The percentile (or percentiles) of distances distribution to be used as a threshold (or thresholds).
    Each percentile must be greater than 0 and smaller than 100. The threshold defines the maximal distance 
    (i.e. maximal dissimilarity) between isolates of the same ST (cluster). 
    
* `find_recommended_thresh` (boolean, optional, default = False): 

    if True, ignore parameter `percentiles` and run the **<em>find recommended threshold</em>** procedure 
    (see 2.3).
    
* `percentiles_to_check` (1-D array-like of floats, optional, default = numpy.arange(.5, 20.5, 0.5)): 

    The percentiles of distances distribution to be evaluated by the **<em>find recommended threshold</em>** procedure 
    (see 2.3). The array must contain at least 2 percentiles; each percentile 
    must be greater than 0 and smaller than 100.

* `simulated_samples` (int, optional, default = 0): 

    The number of samples (clustering structures) to simulate for the computation of the p-value of the observed ARI
    (see 2.2).
    For the significance of the p-values results, it's recommended to use ~1000 samples or more (see 
    [Qannari et al., 2014](https://www.sciencedirect.com/science/article/abs/pii/S0950329313000852)).
    In case `simulated_samples`=0, simulation won't run and p-values won't be calculated.

* `plot_results` (boolean, optional, default = True): 

    If True, plot the ARI and p-value results for each selected threshold as a function of the number of genes.

* `n_jobs` (int, optional, default = min(60, <em>number of CPUs in the system</em>)): 

    The maximum number of concurrently running jobs.

##### Returns:
`analysis_results` (DataFrame):

ARI and p-value (if `simulated_samples` > 0) computed for each subset of most important genes, and for each selected
threshold.
                 
<br>

#### 2.2 ARI simulation study for p.v calculation

P-value is calculated using a significance test of the Adjusted Rand Index (ARI), suggested by 
[Qannari et al., 2014](https://www.sciencedirect.com/science/article/abs/pii/S0950329313000852).
The suggested permutation test involves a simulation of a large number (~1000) of partitions (i.e. clustering 
structures) with several constrains. The number of simulated partitions is set by the input parameter 
`simulated_samples`.

```python
measure = 'shap'
# define the percentiles of distances distribution to be used as thresholds (optional)
percentiles = [0.1, 0.5, 1, 1.5]
# define the number of simulated samples
simulated_samples = 1000
analysis_results = gene_reduction_analysis(data, gi_results, measure, percentiles=percentiles, 
                                           simulated_samples=simulated_samples)
analysis_results.to_csv(join(path, f'analysis_Legionella_pneumophila_{measure}' + '.csv'), index=False)
display(analysis_results)
```

<p align="center">
  <img width="811" src="/docs/analysis_results_pv.png" />
</p>
      
<br>

#### 2.3 Find recommended threshold

This procedure uses an heuristic to find a suitable threshold for generating an induced clustering structure (i.e. STs).
Its search space is the list of percentiles provided as an input parameter `percentiles_to_check`.

To represent the typing performance achieved by a particular threshold, we compute the ARI it results for each
subset of selected genes and construct a vector composed of these ARI elements.

To find a potentially more precise threshold, we run a serial evaluation process starting from the minimal
threshold (baseline) up to the maximal threshold in the search space.
At each iteration, the ARI vector of the 'baseline' threshold is compared with the ARI vector of the 'next'
(second minimal) threshold using a distance function; this function is a "non-absolute" L1 distance, i.e. it equals
to the sum of the differences of two vectors' coordinates, when subtracting the 'baseline' from the 'next'.
In case the distance is positive (i.e. 'next' performs better) the 'next' threshold will be defined as the new
'baseline' and will be compared with the subsequent potential threshold. Otherwise, the search is done and the 
'baseline' is selected as the recommended threshold.

In case `find_recommended_thresh` is True, the analysis results are provided for the recommended threshold merely.

```python
measure = 'shap'
# define the percentiles of distances distribution to be evaluated as thresholds (optional)
search_space = [0.1, 0.3, 0.5, 1, 1.5, 2, 2.5, 3]
# set find_recommended_thresh to True
analysis_results = gene_reduction_analysis(data, gi_results, measure, find_recommended_thresh=True, 
                                           percentiles_to_check=search_space, simulated_samples=1000)
analysis_results.to_csv(join(path, f'analysis_Legionella_pneumophila_{measure}' + '.csv'), index=False)
display(analysis_results)
```

<p align="center">
  <img width="811" src="/docs/analysis_results_find_thresh.png" />
</p>


