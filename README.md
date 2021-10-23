# HINT: Hierarchical Interaction Network for Predicting Clinical Trial Approval Probability

## Table Of Contents 

- Installation
  * Setup conda environment 
  * Activate conda
- Raw Data 
  * clinicaltrial.gov
  * DrugBank
  * MoleculeNet 
- Data Preprocessing 
  * Collect all the NCTIDs
  * diseases to icd10 
  * drug to SMILES 
  * ICD-10 code hierarchy
  * Sentence Embedding for trial protocol 
  * Selection of clinical trial
  * Data split 
  * Generated Dataset and Statistics  
- Learn and Inference 
  * Phase I/II/III prediction
  * Indication prediction 
- Contact 

--- 



## Installation

### Setup conda environment
```bash
conda env create -f conda.yml
```
An alternative way is to build conda environment step-by-step.  
```bash
conda create -n predict_drug_clinical_trial python==3.7 
conda activate predict_drug_clinical_trial 
```

For example, it uses `conda` or `pip` to install the required packages. It may take a long time. 

```bash 
conda install -c rdkit rdkit  
pip install tqdm scikit-learn 
pip install torch
pip install seaborn 
pip install scipy
```


### Activate conda environment
```bash
conda activate predict_drug_clinical_trial
```







## Raw Data 



### ClinicalTrial.gov

We download all the clinical trials records from [ClinicalTrial.gov](https://clinicaltrials.gov/AllPublicXML.zip). 
It contains 348,891 clinical trial records. The data size grows with time because more clinical trial records are added. 
It describes many important information about clinical trials, including NCT ID (i.e.,  identifiers to each clinical study), disease names, drugs, brief title and summary, phase, criteria, and statistical analysis results. 


- output
  - `./raw_data`: store all the xml files for all the trials (identified by NCT ID).  
  - `./trialtrove/trial_outcomes_v1.csv` 


```bash 
mkdir -p raw_data
cd raw_data
wget https://clinicaltrials.gov/AllPublicXML.zip
```


Then we unzip the ZIP file. The unzipped file occupies over 8.6 G. Please make sure you have enough space. 
```bash 
unzip AllPublicXML.zip
cd ../
```

### DrugBank

We use [DrugBank](https://go.drugbank.com/) to get the molecule structures ([SMILES](https://en.wikipedia.org/wiki/Simplified_molecular-input_line-entry_system), simplified molecular-input line-entry system) of the drug. 
The data is saved as `data/drugbank_drugs_info.csv `  


### ClinicalTable

[ClinicalTable](https://clinicaltables.nlm.nih.gov/) is a public API to convert disease name (natural language) into ICD-10 code. 

### MoleculeNet

[MoleculeNet](https://moleculenet.org/) include five datasets across the main categories of drug pharmaco-kinetics (PK). For absorption, we use the bioavailability dataset. For distribution, we use the blood-brain-barrier experimental results provided. For metabolism, we use the CYP2C19 experiment paper, which is hosted in the PubChem biassay portal under AID 1851. For excretion, we use the clearance dataset from the eDrug3D database. For toxicity, we use the ToxCast dataset, provided by MoleculeNet. We consider drugs that are not toxic across all toxicology assays as not toxic and otherwise toxic. 


## Data Preprocessing 


### Collect all the NCTIDs
- input
  - `raw_data/`: raw data, store all the xml files for all the trials (identified by NCT ID).   

- output
  - `data/all_xml`: store NCT IDs for all the xml files for all the trials.  

```bash
find raw_data/ -name NCT*.xml | sort > data/all_xml
```
The current version has 348,891 trial IDs. 


### Disease to ICD-10 code

- description

  - The diseases in [ClinicalTrialGov](clinicaltrials.gov) are described in natural language. 

  - On the other hand, [ICD-10](https://en.wikipedia.org/wiki/ICD-10) is the 10th revision of the International Statistical Classification of Diseases and Related Health Problems (ICD), a medical classification list by the World Health Organization (WHO). It leverages the hierarchical information inherent to medical ontologies. 

  - We use [ClinicalTable](https://clinicaltables.nlm.nih.gov/), a public API to convert disease name (natural language) into ICD-10 code. 

- input 
  - `raw_data/ ` 
  - `data/all_xml`   

- output
  -	`data/diseases.csv ` 

```bash 
python src/collect_disease_from_raw.py
```


### drug to SMILES 

- description
  
  - [SMILES](https://en.wikipedia.org/wiki/Simplified_molecular-input_line-entry_system) is simplified molecular-input line-entry system of the molecule. 

  - The drugs in [ClinicalTrialGov](clinicaltrials.gov) are described in natural language. 

  - [DrugBank](https://go.drugbank.com/) contains rich information about drugs. 

  - We use [DrugBank](https://go.drugbank.com/) to get the molecule structures in terms of SMILES. 

- input
  - `data/drugbank_drugs_info.csv `  

- output
  - `data/drug2smiles.pkl `  

```bash
python src/drug2smiles.py 
```



### Selection of clinical trial

We design the following inclusion/exclusion criteria to select eligible clinical trials for learning. 

- inclusion criteria 
  - study-type is interventional 
  - intervention-type is drug
  - p-value in primary-outcome is available
  - disease codes are available 
  - drug molecules are available 
  - eligibility criteria are available


- exclusion criteria 
  - study-type is observational 
  - intervention-type is surgery, biological, device
  - p-value in primary-outcome is not available
  - disease codes are not available 
  - drug molecules are not available 
  - eligibility criteria are not available


- input    
  - `data/diseases.csv ` 
  - `data/drug2smiles.pkl`  
  - `data/all_xml ` 
  - `trialtrove/*`       


- output 
  - `data/raw_data.csv` (17,592 trials)

The csv file contains following features:

* `nctid`: NCT ID, e.g., NCT00000378, NCT04439305. 
* `status`: `completed`, `terminated`, `active, not recruiting`, `withdrawn`, `unknown status`, `suspended`, `recruiting`. 
* `why_stop`: for completed, it is empty. Otherwise, the common reasons contain `slow/low/poor accrual`, `lack of efficacy`
* `label`: 0 (failure) or 1 (approved).  
* `phase`: I, II, III or IV. 
* `diseases`: list of diseases. 
* `icdcodes`: list of icd-10 codes.
* `drugs`: list of drug names
* `smiless`: list of SMILES
* `criteria`: egibility criteria 


```bash
python src/collect_raw_data.py | tee data_process.log 
```





<p align="center"><img src="./figure/dataset.png" alt="logo" width="650px" /></p>




### Data Split 

- Split criteria
  - phase I: phase I trials, augmented with phase IV trials as positive samples. 
  - phase II: phase II trials, augmented with phase IV trials as positive samples.  
  - phase III: phase III trials, augmented with failed phase I and II trials as negative samples and successed phase IV trials as positive samples. 
  - indication: trials that fail in phase I or II or III are negative samples. Trials that pass phase III or enter phase IV are positive samples.  

- input
  - `data/raw_data.csv` 

- output: 
  - `data/phase_I_{train/valid/test}.csv` 
  - `data/phase_II_{train/valid/test}.csv` 
  - `data/phase_III_{train/valid/test}.csv` 
  - `data/indication_{train/valid/test}.csv` 


```bash
python src/data_split.py 
```


### 4.2 ICD-10 code hierarchy 

- input
  - `data/raw_data.csv` 

- output: 
  - `data/icdcode2ancestor_dict.pkl` 


```bash 
python src/icdcode_encode.py 
```

### 4.3 sentence embedding 


- input
  - `data/raw_data.csv` 

- output: 
  - `data/sentence2embedding.pkl` 


```bash 
python src/protocol_encode.py 
```


### 4.4 Data Statistics 

| Dataset  | \# Train | \# Valid | \# Test | \# Total | Split Date |
|-----------------|-------------|-------------|------------|-------------|------------|
| Phase I |  1028  |  146  |  295  |   1469  |  08/13/2014  | 
| Phase II | 2667 |  381 | 762  |   3810  |  03/20/2014  | 
| Phase III |  4286  |  612  |  1225 |  6123  |  04/07/2014  | 
| Indication |  3767  |  538  |  1077   |  5382  |  05/21/2014  | 

We use temporal split, where the earlier trials (before split date) are used for training and validation, the later trials (after split date) are used for testing. The train:valid:test ratio is 7:1:2. 





























## 5. Learn and Inference 


After processing the data, we learn the Hierarchical Interaction Network (HINT) on the following four tasks. The following figure illustrates the pipeline of HINT. 

<p align="center"><img src="./figure/hint.png" alt="logo" width="790px" /></p>



### Phase level Prediction

```bash
python src/learn_phaseI.py
```


```bash
python src/learn_phaseII.py
```


```bash
python src/learn_phaseIII.py
```



### Indication Prediction

```bash
python src/learn_indication.py 
```





### METRICS

- **PR-AUC** (Precision-Recall Area Under Curve). Precision-Recall curves summarize the trade-off between the true positive rate and the positive predictive value for a predictive model using different probability thresholds.
- **F1**. The F1 score is the harmonic mean of the precision and recall.
- **ROC-AUC** (Area Under the Receiver Operating Characteristic Curve). ROC curve summarize the trade-off between the true positive rate and false positive rate for a predictive model using different probability thresholds. 


### Result 

The empirical results are given for reference. The mean and standard deviation of 5 independent runs are reported. 

| Dataset  | PR-AUC | F1 | ROC-AUC |
|-----------------|-------------|-------------|------------|
| Phase I | 0.7406 (0.0221) | 0.8474 (0.0144) |  0.8383 (0.0186) |    
| Phase II | 0.6030 (0.0198) | 0.7127 (0.0163) | 0.7850 (0.0136)  |    
| Phase III | 0.6279 (0.0165) | 0.6419 (0.0183) | 0.7257 (0.0109) |    
| Indication | 0.7136 (0.0120) | 0.7798 (0.0087) | 0.7987 (0.0111)  |   




### Jupyter Notebook Tutorial 

Please see `learn_phaseI.ipynb` for details. 










## Contact

Please contact futianfan@gmail.com for help or submit an issue. This is a joint work with [Kexin Huang](https://www.kexinhuang.com/), [Cao(Danica) Xiao](https://sites.google.com/view/danicaxiao/), Lucas M. Glass and [Jimeng Sun](http://sunlab.org/). 

























