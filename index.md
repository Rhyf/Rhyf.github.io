Skip to content
Search or jump to…
Pull requests
Issues
Marketplace
Explore
 
@Rhyf 
Rhyf
/
METAFlux
Public
Code
Issues
Pull requests
Actions
Projects
Wiki
Security
Insights
Settings
METAFlux/vignettes/intro_metaflux.Rmd
@Rhyf
Rhyf Add files via upload
Latest commit a299b0b 11 days ago
 History
 1 contributor
426 lines (266 sloc)  18 KB

---
title: "Tutorial of METAFlux:Bulk and single cell RNA-seq"
output:
  prettydoc::html_pretty:
    theme: cayman
    highlight: github
    toc: yes
toc-title: Table of Contents
vignette: |
  %\VignetteIndexEntry{Tutorial of METAFlux:Bulk and single cell RNA-seq} %\VignetteEngine{knitr::knitr} %\VignetteEncoding{UTF-8}
---

# 1.Introduction
This vignette will cover a quick workflow and a step by step breakdown of calculating metabolic fluxes using METAFlux(METAbolic Flux balance analysis), a framework to obtain metabolic fluxes from bulk RNA-seq or single-cell RNA-seq data. Fluxes, usually measured in mmol/gDW/hr, are velocities of metabolic reactions. In METAFlux, however, the predicted fluxes are calculated and normalized from gene expression data, meaning that the results are relative flux scores.

# 2.Quick workflow

## 2.1 For bulk RNA-seq sample

We have run both cell line and human sample models, but users will only need to run one model by choosing only **one** medium file as the input to run METAFlux. 

**1.cell line data**
```{R, message=FALSE,results='hide'}
library(METAFlux)
#load the toy example
data("bulk_test_example")
#medium file for cell line model
data("cell_medium")
#Calculate mras for cell line data
scores<-calculate_reaction_score(bulk_test_example)
#Calculate flux for cell line data
flux<-compute_flux(mras=scores,medium = cell_medium) 
```


**2.human derived data**
```{R, message=FALSE,results='hide'}
library(METAFlux)
#load the toy example
data("bulk_test_example")
# medium file for human derived samples
data("human_blood")
#Calculate mras for human sample data
scores<-calculate_reaction_score(bulk_test_example)
#Calculate flux for human sample data
flux<-compute_flux(mras=scores,medium=human_blood) 
```


## 2.1 For sc RNA-seq sample

Here we only demonstrate how to interface with Seurat object for  one **patient sample**. Please refer to the example code at the end of this section for multiple samples pipeline.



```{r,message=FALSE}
library(METAFlux)
#load the toy seurat example, this data only has one patient. For mutiple samples,see example code in section 4
data("sc_test_example")
# medium file for human derived samples
data("human_blood")
#extract the normalized data from seurat obejct
countdata<-as.data.frame(sc_test_example@assays$RNA@data)
#replace the colnames with cluster/cell type information,here we will model fluxes on cell-type level
data.table::setnames(countdata,sc_test_example$cell_type)
#check the colnames, it has 43 t cells and 300 tumor cells. Make sure the colnames are correct.
table(colnames(countdata))
#Calculate the mean expression for bootstrap samples
#For the sake of demonstration, we only set the number of bootstraps to 3. 
#In real analysis, the number of bootstraps should be much higher(e.g. 100, 200....)
#here this input "countdata" is a dataframe
mean_exp=calculate_avg_exp(raw=countdata,n_bootstrap=3,seed=1)
#calculate metabolic reaction scores
scores<-calculate_reaction_score(data=mean_exp)
#calculate the fractions of celltype/clusters
round(table(sc_test_example$cell_type)/nrow(sc_test_example@meta.data),3)
#calculate flux: here we used human blood as our medium, but please use cell line medium when samples are cell line.
```

```{r,message=F,results='hide'}
#num_cell=number of cell types/clusters, here we only have t cells and tumor cell, thus input is 2
flux=compute_sc_flux(num_cell = 2,fraction =c(0.125,0.875),fluxscore=scores,medium = human_blood)
```


<details>
  <summary>**Pipeline for multiple samples/patients**</summary>
```{r regressvar, fig.height=7, fig.width=11, results='hide',eval = FALSE}
#load your seurat object
#sample code for 4 clusters in multiple patients
obj.list <- SplitObject(seurat, split.by = "Sample")
for (i in c(1:length(obj.list))){
sc<-obj.list[[i]]
countdata<-as.data.frame(sc@assays$RNA@data)
data.table::setnames(countdata,sc$Cell_type)
# print(table(colnames(countdata)))
mean_exp=calculate_avg_exp(raw=countdata,n_bootstrap=100,seed=1)
scores<-calculate_reaction_score(data=mean_exp)
#g=round(table(sc$Cell_type)/nrow(sc@meta.data),3)
g=table(sc$Cell_type)/nrow(sc@meta.data)
print(g)
flux=compute_sc_flux(num_cell = 4,fraction =c(g[1],g[2],g[3],g[4]),fluxscore=scores,medium = human_blood)
saveRDS(flux,paste0("object",i,".rds"))
}
```
</details>
\ 






# 3.Step by step breakdown:Bulk RNA-seq pipeline
## 1.Load the library
```{r}
library(METAFlux)
```

## 2.Load data
### 2.1 Load gene expression data.

METAFlux requires gene expression data as an input. 

 + The gene expression matrix should be gene x sample matrix where row names are **human gene names** (gene symbols), and column names should be sample names. Please note that METAFlux does not support other gene IDs.
 + The input gene expression matrix should be **normalized** (e.g., log-transformed, etc.) before using METAFlux. METAflux will not perform any normalization on expression data.
 + Gene expression data **cannot** have negative values.

```{r}
#load our toy example data
data("bulk_test_example")
head(bulk_test_example)
```


### 2.2 Load the METAFlux underlying GEM information.

Before we get started, we can take a look at Human-GEM file.We used Human-GEM[1] (consisting of 13082 metabolic reactions and 8378 metabolites)as our underlying metabolic model. For each reaction, we have one unique **Reaction ID** and **SUBSYSTEM** (i.e. which pathway this reaction belongs to).

```{r}
data("human_gem")
```



Take a quick look at the first 6 rows.
```{r,echo=FALSE, out.width = '100%'}
library(kableExtra)
head(human_gem[,c(2:6,12)],6)%>%
kbl() %>%
kable_classic_2(full_width = F)
```


For each reaction, we also get other important information from this file such as: whether the reaction is reversible, which compartment it is taking place in, and what metabolites and genes are involved. The 1 to 1 relationship is color coded in the cartoon below.
![](humangendemo.jpeg)

### 2.3 Load the METAFlux medium.

```{r}
data("cell_medium")# for cell line model
data("human_blood")# for human derived samples
```


Take a quick look at the first 6 rows of cell_medium.
```{r,echo=FALSE, out.width = '100%'}
library(kableExtra)
head(cell_medium)%>%
kbl() %>%
kable_classic_2(full_width = F)
```

#### 2.3.1 But what is 'medium' and why do we need it？

In METAFlux, ‘medium’ reflects the nutrients(metabolites) in the  sample tumor microenvironment (TME) and those nutrients can be allowed to uptake into cells. 

<details>
  <summary>**Option 1: If users do not have prior knowledge about medium composition.**</summary>


Here, we provide 2 general mediums if you have no prior knowledge about your medium. There are 2 general medium files in METAFlux: **cell_medium** and **human_blood medium**. Cell_medium can be used in cell line models, and human blood medium can be used in patient-derived samples. 

The cell line medium contains 44 metabolites[1], and their corresponding exchange reactions can be found under column ‘reaction_name.’The blood medium contains 64 metabolites in human blood. 
</details>
\  


<details>
  <summary>**Option 2: If users have prior knowledge about their medium composition.**</summary>




+ To delete nutrients, simply delete the entire row where that nutrient is located, but please do not change the column names or delete columns. 

+ To add additional nutrients, please refer to the **nutrient lookup files**.

```{r}
data("nutrient_lookup_files")
```

‘Nutrient look up files’ contains 1648 exchange reactions(*note:exchange reactions are artificial reactions! They are included here because this is a mathematical representation of uptake/secrete metabolites into the extracellular space*) that describe how cells uptake or release metabolites. For example, if we need to add metabolite “naphthalene,” one could search in the search box, and the corresponding nutrient reaction and the equation will be located. Next, one needs to add new row of naphthalene and HMR_7110 to the current medium file. This updated medium file will later be used for modeling.
![](adddemo.jpeg)
</details>
\



## 3.Calculate  MRAS (Metabolic Reaction Activity Score)
Next, we can calculate a single sample normalized MRAS from gene expression using GPR(Gene-protein-reaction).
```{r}
scores<-calculate_reaction_score(bulk_test_example)
head(scores)
```

## 4.Calculate flux
Subsequently, we calculate the metabolic fluxes for the 13082 reactions. 


```{r,results='hide'}
flux<-compute_flux(mras=scores,medium = cell_medium)#if data are cell line samples
flux<-compute_flux(mras=scores,medium = human_blood)#if data are human derived samples
```




## 5.Inspecting and interpreting the flux data

+ Signs of fluxes are biologically meaningful.
Here “sign” of flux indicates the direction. For example, for the nutrient uptake/release case (1648 exchange reactions in **nutrient lookup file**), a positive value means a release of compounds, and a negative value means uptake of compounds. For other reactions, a positive value implies the net flux is in a forward direction, and a negative flux indicates the net flux is in a backward direction. Absolute values denote the magnitude of flux.

+ One thing to note is that since we are minimizing the sum of overall fluxes in the model, we will get a parsimonious flux data output, meaning lots of reactions will approach 0 flux. (For example, some reactions should only go forward, but the predicted flux has a negative sign with a very small value and can be regarded as approaching 0 flux)

+ Extraction of targeted data.
If one is strictly interested in metabolite uptake or release, one can search “nutrient lookup file” to obtain the metabolite exchange Reaction ID. For example, if we want to know the glucose uptake, we need to search for ‘glucose’ to get the glucose uptake reaction, which is HMR_9034. Those can be regarded as glucose metabolite uptake rates. Next, we can pull out the data:


```{r}
data("nutrient_lookup_files")
glucose<-data.frame(glucose=flux[grep("HMR_9034",human_gem$ID),])
glucose
```

If one is interested in other reactions (e.g., glycolysis, oxphos, etc.), one needs to search for Reaction_ID using the human_gem file. For example, if we are interested in reaction HMR_4363 in glycolysis pathway:

```{r}
#HMR_4363: 2-phospho-D-glycerate[c] <=> H2O[c] + PEP[c]
HMR_4363<-data.frame(hmr4363=flux[grep("HMR_4363",human_gem$ID),])
```






# 4.Step by step breakdown: Single-cell pipeline
We will model the whole tumor microenvironment as one community to account for the metabolic interaction between groups. Here, we do not compute cell-wise flux. Instead, we model fluxes at cluster/cell-type level. We will demonstrate use cases using Seurat object.

**To estimate metabolic fluxes, we need to provide:**

1. Gene by cell expression matrix. The input gene expression matrix should be normalized (e.g., log-transformed, etc.) before using METAFlux. METAflux will not perform any normalization on expression data.
2. Cluster or cell type assignments for each cell.
3. Cluster/cell type fractions$*$. These proportions should be retrieved from experiments or calculated from matched bulk data using CIBERSORTx[2]. However, most datasets do not have such information. Directly calculated cluster (group) fractions in single-cell data could still be used. However, further studies are warranted to evaluate the findings since those proportions may deviate from the truth due to bias introduced during tissue dissociation .


$*$ Usually, each single cell data contains more than one subject (patients). **It would be more natural to estimate the proportions of each cluster/cell type separately for each subject**, as METAFlux will be modeling the interaction within the same TME. However, depending on specific biological questions and data quality, one can still study the average effect across subjects (i.e., proportions of each cluster/cell type are estimated using all data). 



## 1. Load library and data files
```{r}
library(METAFlux)
#load medium file.Please read 'bulk step by step breakdown section 2.3' for more details
data("human_blood")
data("cell_medium")
#load underlying GEM.Please read 'bulk step by step breakdown section 2.2' for more details
data("human_gem")
```


## 2.	Load the single cell data and set the cell type or cluster assignment as its column names.

Please make sure the column names are appropriately named in such a way. The key point is to make sure the colnames in the **countdata** are in the correct format like ‘T_cells","T_cells","tumor'..... 
```{r}
data("sc_test_example")#seurat toy example,this data contains only one patient
#extract the normalized data from seurat obejct
countdata<-as.data.frame(sc_test_example@assays$RNA@data)
#replace the colnames with cluster/cell type information,here we will model fluxes on cell-type level
data.table::setnames(countdata,sc_test_example$cell_type)
colnames(countdata)[1:10]#Make sure the colnames are correct!!!
#check the colnames, it has 43 t cells and 300 tumor cells. 
table(colnames(countdata))
```

## 3.	Create an average expression profile for stratified bootstrapped samples for this patient
```{r}
#Calculate the mean expression for bootstrap samples
#Here this input "countdata" is a dataframe!
#For the sake of demonstration, we only set the number of bootstraps to 3. 
#In real analysis, the number of bootstraps should be much higher(e.g. 100, 200....)
mean_exp=calculate_avg_exp(raw=countdata,n_bootstrap=3,seed=1)
head(mean_exp)
```

## 4.	Calculate MRAS(Metabolic Reaction Activity Score)
We can calculate single sample normalized MRAS from data using Gene-protein-reaction (GPR).This step is the same with the bulk RNA-seq pipeline. 
```{r}
#calculate metabolic reaction scores
scores<-calculate_reaction_score(data=mean_exp)
```


Take a look at MRAS results
```{r}
head(scores,3)
```


## 5. Compute flux
We calculate the metabolic fluxes. Note that the order of *fraction* parameters input should be consistent with the order in **scores** results. For example, Cell type 1(T cells) is the first column of **scores**, and the cell type 2 (tumor cells) is the second column of **scores**, etc. More importantly, fractions need to sum up to 1.


```{r}
#calculate the fractions of celltype/clusters
round(table(sc_test_example$cell_type)/nrow(sc_test_example@meta.data),3)
#calculate flux,here we used human blood as our medium, but please change to cell line medium if you sample is from cell line.
#num_cell:number of cell types/clusters=2
flux=compute_sc_flux(num_cell = 2,fraction =c(0.125,0.875),fluxscore=scores,medium = human_blood)
```



## 6.Inspecting and interpreting the flux data

+ The total dimension of predicted flux data will be $(num\_cell*13082+1648)*number\_of\_bootstrap$.
Row names are annotated with celltype 1, celltype2, etc., to indicate which cell type/cluster this reaction is referring. 
Row names with “external_medium” are reactions that the whole TME exchanges (uptake or release) metabolites with the external environment. Those reactions refer to the entire tumor community, not one specific cell type.
Row names with “internal_medium “are reactions that specific cell type/cluster exchange (uptake or release) metabolites with TME.
```{r}
dim(flux)
#(2*13082+1648)*3=27812*3
```




+ Signs of fluxes are biologically meaningful.
This is already mentioned in the bulk pipeline, but here the logic is the same. For the nutrient uptake/release case (those represented by 1648 exchange reactions in nutirent lookup files), a positive value means a release of compounds, and a negative value means uptake of compounds. For other reactions, a positive value implies the net flux is in a forward direction, and a negative flux indicates the net flux is in a backward direction. Absolute values denote the magnitude of flux

+ The cell type-specific fluxes are average flux. The cell type-specific fluxes are mean flux or unit flux for each cell type. More specifically, if cell type 1 flux of reaction *A* equals 0.2, this simply means the flux of reaction *A* for an average cell in cell type 1 is 0.2. The external_medium fluxes are **not** average flux, but instead, they are weighted total fluxes.The fluxes between external_medium and internal_medium hold such an equation for exchange reactions:

$Proportion^{cell type1}\cdot Intenal_-Medium^{Celltype1}_{exchange_-reaction_i}+ Proportion^{cell type2} \cdot Intenal_-Medium^{Celltype2}_{exchange_-reaction_i}+……..=Entenal_-Medium_{exchange_-reaction_i}$

+ We can extract the reactions we are interested. Again, if one is strictly interested in metabolite uptake or release, one can search “nutrient lookup file” to obtain the metabolite exchange Reaction ID( Please read 'bulk step by step breakdown section 2.3.1' for more details). For example, if we want to know the glucose uptake, we need to search for ‘glucose’ to get the glucose uptake reaction, which is HMR_9034. Next, we can pull out the data by,


```{r}
data("nutrient_lookup_files")
glucose<-data.frame(glucose=flux[grep("HMR_9034",rownames(flux)),])
glucose
```

Taken all those key points together, we see Glucose.V1, Glucose.v2, Glucose.V3 are 3 bootstrap samples. One may look at the distribution of all bootstraps to compare T cells and tumor cells nutrient uptake profile.

+ “celltype 1 internal_medium HMR_9034" is the glucose uptake flux for cell type 1(referring to T cells in toy example).
+ "celltype 2 internal_medium HMR_9034" is the glucose uptake flux for cell type 2（referring to Tumor cells in toy example).
+ ”external_medium HMR_9034" is the glucose uptake flux for the whole tumor community（referring to combined T cell and tumor cell community in toy example).



# 5.More relevant readings

Readings for metabolic flux balance analysis and GEM(Genome-scale metabolic models):

1. [What is flux balance analysis?](https://www.nature.com/articles/nbt.1614)

2. [An atlas of human metabolism](https://www.science.org/doi/10.1126/scisignal.aaz1482)


# 6.References

[1]Robinson, J.L., et al., An atlas of human metabolism. Sci Signal, 2020. 13(624).

[2]Newman, A.M., Steen, C.B., Liu, C.L. et al. Determining cell type abundance and expression from bulk tissues with digital cytometry. Nat Biotechnol 37, 773–782 (2019). https://doi.org/10.1038/s41587-019-0114-2
Footer
© 2022 GitHub, Inc.
Footer navigation
Terms
Privacy
Security
Status
Docs
Contact GitHub
Pricing
API
Training
Blog
About
You have no unread notifications
