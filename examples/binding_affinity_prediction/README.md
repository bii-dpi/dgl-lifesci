# Binding Affinity Prediction

## Datasets
- **PDBBind**: The PDBBind dataset in MoleculeNet [1] processed from the PDBBind database. The PDBBind 
database consists of experimentally measured binding affinities for bio-molecular complexes [2], [3]. 
It provides detailed 3D Cartesian coordinates of both ligands and their target proteins derived from 
experimental(e.g., X-ray crystallography) measurements. The availability of coordinates of the 
protein-ligand complexes permits structure-based featurization that is aware of the protein-ligand 
binding geometry. The authors of [1] use the "refined" and "core" subsets of the database [4], more carefully 
processed for data artifacts, as additional benchmarking targets.

## Models
- **Atomic Convolutional Networks (ACNN)** [5]: Constructs nearest neighbor graphs separately for the ligand, protein and complex 
based on the 3D coordinates of the atoms and predicts the binding free energy.

- **PotentialNet** [6]: A 3-stage model that combines graph convolutional neural network (GCNN) with molecular graphs and KNN graphs and fully connected neural network (FCNN). The model consists of three main steps: (1) covalent-only propagation, (2) dual noncovalent and covalent propagation, and (3) ligand-based graph gather.
    1. Stage 1. Both ligand and protein graphs are constructed and featurized based on covalent information using `dgllife.utils.CanonicalAtomFeaturizer` and `dgllife.utils.CanonicalBondFeaturizer` so that only chemically bonded atoms are connected in the graph. The feature propagation is passed onto a multi-step gated recurrent unit (GRU) followed by a linear layer with sigmoid activation.
    2. Stage 2. A new pair of knn-graphs of ligand and protein is constructed from 3-D coordinates of the molecules. The edge type between atoms is a combination of covalent bond types and their physical distances. The output from stage 1 is used as initial features. The feature propagation is again passed onto a multi-step gated recurrent unit (GRU) followed by a linear layer with sigmoid activation.
    3. Stage 3. Feature gathering is performed only on ligand atoms in graphs from stage 2. The final prediction is computed from feature propagation through a multi-layer fully connected neural network with ReLU activation.


## Usage

Use `main.py` with arguments

* `-m`, Model to use. Available: `{ACNN, PotentialNet}`.
* `-v`, Version of PDBBind dataset. Available: `{v2007, v2015}`.
* `-d`, PDBBind subset (core or refined), and splitting method. Currently implemented: 
    ```
    {PDBBind_core_pocket_random, 
    PDBBind_core_pocket_scaffold,
    PDBBind_core_pocket_stratified, 
    PDBBind_core_pocket_temporal,
    PDBBind_refined_pocket_random, 
    PDBBind_refined_pocket_scaffold,
    PDBBind_refined_pocket_stratified, 
    PDBBind_refined_pocket_temporal,
    PDBBind_refined_pocket_structure, 
    PDBBind_refined_pocket_sequence}
    ```   
    * Note that `structure` refers to "Agglomerative Structure Split" provided by [6], and is only implemented for PDBBind v2007 Refined set; `sequence` refers to "Agglomerative Sequence Split" provided by [6], and is only implemented for PDBBind v2007 Refined set.
* `--pdb_path`, local path of existing PDBBind dataset. Specify this argument to a local path of **customized dataset**, which should follow the structure and the naming format of PDBBind v2015.
* `--test_on_core`, bool, whether to use the whole core set as test set when training on refined set, default True.
* `--save_r2`, path to save r2 at each epoch, default not save.
* `--num_workers`, number of workers for Dataloader, default to 8.
* `-t`, int, number of trials to run, default to 1.

### For access to more hyperparameters, see `./configure.py`.

#### PotentialNet Model Parameters

* `f_in`: int. 
    The dimension size of input features to GatedGraphConv, 
    equivalent to the dimension size of atomic features in the molecular graph.
* `f_bond`: int. 
    The dimension size of the output from GatedGraphConv in stage 1,
    equivalent to the dimension size of input to the linear layer at the end of stage 1.
* `f_spatial`: int. 
    The dimension size of the output from GatedGraphConv in stage 2,
    equivalent to the dimension size of input to the linear layer at the end of stage 2.
* `f_gather`: int. 
    The dimension size of the output from stage 1 & 2,
    equivalent to the dimension size of output from the linear layer at the end of stage 1 & 2.
* `n_etypes`: int. 
    The number of heterogeneous edge types for stage 2.
    Currently implemented as 5(the number of covalent bond types in stage 1) + the number of distance bins in stage 2.
* `n_bond_conv_steps`: int. 
    The number of bond convolution layers(steps) of GatedGraphConv in stage 1.
* `n_spatial_conv_steps`: int. 
    The number of spatial convolution layers(steps) of GatedGraphConv in stage 2.
* `n_rows_fc`: list of int. 
    The widths of the fully connected neural networks at each layer in stage 3.
* `dropouts`: list of 3 floats. 
    The amount of dropout applied at the end of each stage.

## Performance

### PDBBind

#### ACNN

| Subset  | Splitting Method | Test MAE | Test R2 |
| ------- | ---------------- | -------- | ------- |
| Core    | Random           | 1.7688   | 0.1511  |
| Core    | Scaffold         | 2.5420   | 0.1471  |
| Core    | Stratified       | 1.7419   | 0.1520  |
| Core    | Temporal         | 1.9543   | 0.1640  |
| Refined | Random           | 1.1948   | 0.4373  |    
| Refined | Scaffold         | 1.4021   | 0.2086  |
| Refined | Stratified       | 1.6376   | 0.3050  |
| Refined | Temporal         | 1.2457   | 0.3438  |

#### PotentialNet

| Training Set | Test Set| Splitting Method | PDBBind version | Train R2 (std) | Validation R2 (std) | Test R2 (std) |
| ------------ | --------| ---------------- | --------------- | -------------- | ------------------- | ------------- |
| Refined      | Core    | Random           | v2007           | 0.9305 (0.0106)| 0.4161 (0.0217)     |0.8980 (0.0102)|
| Refined      | Core    | Random           | v2015           | 0.8934 (0.0112)| 0.4227 (0.0170)     |0.8117 (0.0135)|
| Refined (core removed)| Core | Random     | v2007           | 0.6801 (0.0359)| 0.4605 (0.0185)     |0.4086 (0.0159)|


The results are computed over 100 trials and R2 from the best epoch is reported. 

##### Reported Performance from PotentialNet paper [6]

| Training Set           | Test Set| PDBBind version | Test R2 (std) |
| ---------------------- | ------- | --------------- | ------------- |
| Refined (core removed) | Core    | v2007           | 0.668 (0.043) |

## Speed

### ACNN

Comparing to the [DeepChem's implementation](https://github.com/joegomes/deepchem/tree/acdc), we achieve a speedup by 
roughly 3.3 for training time per epoch (from 1.40s to 0.42s). If we do not care about 
randomness introduced by some kernel optimization, we can achieve a speedup by roughly 4.4 (from 1.40s to 0.32s).

## References

[1] Wu et al. (2017) MoleculeNet: a benchmark for molecular machine learning. *Chemical Science* 9, 513-530.

[2] Wang et al. (2004) The PDBbind database: collection of binding affinities for protein-ligand complexes 
with known three-dimensional structures. *J Med Chem* 3;47(12):2977-80.

[3] Wang et al. (2005) The PDBbind database: methodologies and updates. *J Med Chem* 16;48(12):4111-9.

[4] Liu et al. (2015) PDB-wide collection of binding data: current status of the PDBbind database. *Bioinformatics* 1;31(3):405-12.

[5] Gomes et al. (2017) Atomic Convolutional Networks for Predicting Protein-Ligand Binding Affinity. *arXiv preprint arXiv:1703.10603*.

[6] Feinberg et al. (2018) PotentialNet for molecular property prediction. *ACS central science* 4.11: 1520-1530.
