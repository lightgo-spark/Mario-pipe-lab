# Iris Dataset Analysis

pipeline analysis Iris Plants Database (R.A. Fisher, 1936).

## Pipeline

`Extract → Transform → Load → Analysis → Modeling`

- **Extract**: raw CSV loading (no header) from `iris.data` and `bezdekIris.data`
- **Transform**: numeric conversion, whitespace strip, duplicate removal, IQR outlier detection, categorical encoding
- **Load**: cleaned DataFrames for analysis
- **Analysis**: class distribution, descriptive statistics, correlation, scatter/box plots, UCI vs corrected dataset comparison
- **Modeling**: stratified 80/20 split, k-NN (k=5), decision tree (depth=3), confusion matrices, PCA projection

## Usage

```bash
# Notebook (recommended)
jupyter notebook iris_analysis.ipynb
```

## Requirements

Python 3.9+, see `requirements.txt`.

## Key Findings

- Setosa is linearly separable; versicolor/virginica overlap slightly
- Petal features are the key discriminators (class correlation > 0.95)
- Test accuracy: k-NN 100%, decision tree 96.7% (stratified 80/20, seed=42)
- PCA with 2 components captures ~97-98% of variance
