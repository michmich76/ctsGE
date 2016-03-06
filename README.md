# ctsGE - Clustering of Time Series Gene Expression data

 **ctsGE** is a package written for **R**, that preforms clustering of time-series gene expression data.
 Clustering is done by a "structure-based" dissimilarity concept (see, e.g., [Lin and Li 2009](http://dl.acm.org/citation.cfm?id=1561679), [Corduas 2010](http://link.springer.com/chapter/10.1007/978-3-642-03739-9_40)).
 Genes that share the same pattern of expression along all time points, will be grouped together.
 The clustering is done in two steps, we first define an expression profile for each gene, and then cluster each one of these profiles with **kmeans**.
 **ctsGE** package also provides the option to visualize the genes expression pattern in line graphs, made with **ggplot2** package.
```{r, echo = FALSE, message = FALSE}
library(ctsGE)
library(pander)
```




## Workflow of clustering with ctsGE
1. Defining the expression profiles:

    a) For each gene, we calculate its standardized value at all time points by `ctsGE::MadScale`: $$ GEst = \frac{GE - median(GE)}{mad(GE)}$$  
          
      >The user can choose to do the standardization with **scale** (***R*** *function*)  
    b) Next we convert the new standardized values to **0 / 1 / -1**:  
          
            0  when gene expression at a specific time point wasn't significantly different from the other time points
            1  when gene expression at a specific time point was significantly greater from the other time points
           -1 when gene expression at a specific time point was significantly smaller from the other time points  
     
        > The signification level is determined by the user with the `cutoff` option in the `PreparingTheProfiles` function.

2. Clustering each profile with **kmeans** function: $$kmeans(genesInProfile,k)$$

## Installing ctsGE
```{r,eval=FALSE,warning=FALSE,message=FALSE}
install.packages("ctsGE_0.1.tar.gz",repos=NULL,type="source")

```

## Input data and preparations
As input, the **ctsGE** package expects count data as obtained, e. g., from RNA-Seq or another high-throughput
sequencing experiment or micoarray expiriment, in the form of a rectangular table of integer values. The table cell in the
i-th row and the j-th column of the table tells how many reads have been mapped to gene i in time-point j. Analogously, for other types of assays, the time-points of the table might correspond e. g. to different tretment or conditions. In this vignette, we will work with, gene expression profiling in Gene expression in Cryptosporidium parvum-infected human ileocecal adenocarcinoma cells (HCT-8) (Deng et al., 2004^[Deng, M., Lancto, C. A. and Abrahamsen, M. S. (2004). Cryptosporidium parvum regulation of human epithelial cell gene expression. Int. J. Parasitol. 34, 73–82.]), an expression profiling by array extracted from the [Gene Expression Omnibus (GEO)](http://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE2077). For tutorial purpose and for simplification reason, we used only one replicaste out of three, overall six time-points.

```{r,message=FALSE, warning=FALSE}
library(GEOquery)
gse2077 <- getGEO('GSE2077', GSElimits = c(1,6), GSEMatrix = FALSE) 
gseList  <- lapply(GSMList(gse2077),function(x){Table(x)}) # list of the time series tables
```

***Sample accession numbers:***

```{r,message=FALSE, warning=FALSE}
names(gseList)
```
##  Building the ctsGE Object  
*ctsGEList()* is the function that coverts the count matrix into an ctsGE object. First, we create a group variable that tells ctsGE with samples belong to which group and supply that to *ctsGEList* in addition to the count matrix. We can then see the elements that the object contains by using the *names()* function. These elements can be accessed using the $ symbol
```{r,message=FALSE,warning=FALSE}

rts <- readTSGE(gseList,labels = c("0h","6h","12h","24h","48h","72h")) 
names(rts)
rts$timePoints
head(rts$samples)
head(rts$tags)
```

` head(rts$tsTable)`

```{r, echo=FALSE,results='asis'}

panderOptions("table.style","rmarkdown")
pander(head(rts$tsTable))

```

## Defining the expression profiles  
We standardized the counts with [`ctsGE::MadScale`](#workflow-of-clustering-with-ctsge) (default), otherwise it's done with `scale`.  
The profiling to **0 / 1 / -1** as describe [here](#workflow-of-clustering-with-ctsge), determine by the `cutoff` value:  
After standardization, `cutoff` defines the degree of change in gene expression level throughout the given time points.  
How much does the gene expression value distance from the `median` of the gene, in a certain time point. The distance from the `median` is measured in `mad`(median absolute deviation) units, in case of using `scale` method the distance is from the `mean` in `sd` units.  
The user can decide what is the degree, in which the expression level, is not significant enough or not informative enough for their experiment purpose.  For example: if `cutoff` is set to 0.7, genes that their standardized value falls between *0.7 to -0.7* will get the new value **0** because their expression level was not significantly different from other time points expression level.  We recommand to check different `cutoff` values and see which one provides profiles that representive of any expression pattern of your data.

###Checking different cutoffs  

```{r,message=FALSE,warning=FALSE,results="asis"}
raw.counts <- head(rts$tsTable)
scaled.counts <- MadScale(raw.counts)
cutoffs <- seq(0.5,1.5,0.1)
check.list <- lapply(cutoffs,function(i){apply(apply(scaled.counts,2,index,i),1,paste,collapse="")})
check.matrix <- do.call("cbind",check.list); colnames(check.matrix) <- cutoffs
```
**The `raw.counts` for check**
```{r,message=FALSE,echo=FALSE,warning=FALSE,results="asis"}

panderOptions("table.style","simple")
pander(raw.counts)
```

**The `scaled.counts` for check**

```{r,message=FALSE,echo=FALSE,warning=FALSE,results="asis"}

panderOptions("table.style","simple")
pander(scaled.counts)
```

**Profiles of different `cutoffs`:**  
*you can see when cutoffs is between 1.1 to 1.5 the profiles doesn't changes from one to another.*

```{r,message=FALSE,echo=FALSE,warning=FALSE,results="asis"}

panderOptions("table.style","simple")
pander(check.matrix)
```

###Preparing the profiles for our data  
*We chosen to continue with* **`cutoff = 0.8`**
```{r, message=FALSE,warning=FALSE}
prts <- PreparingTheProfiles(x = rts, cutoff = 0.8, mad.scale = TRUE) 
names(prts)
```
 

***Gene expression after standardization:***
` head(prts$scaled) ` 

```{r,message=FALSE,echo=FALSE,warning=FALSE,results="asis"}

panderOptions("table.style","simple")
pander(head(prts$scaled)) 
```


***Gene expression after profiling with `cutoff = 0.8`:***
` head(prts$profile) `
```{r,message=FALSE,echo=FALSE,warning=FALSE,results="asis"}

panderOptions("table.style","simple")
pander(head(prts$profiles)) 
```

## Clustering each profile with kmeans  
The clustering is done with `kmeans`, in order to choose the right k (number of clusters) such that the clusters are compact and well separated, we compute ratio of the within cluster sum of squares (WSS) to the total sum of squares (TSS).  
when: $\frac{WSS}{TSS} < 0.2$ this is the optipal k.  
  
*If you want to cluster a specific profile and choose the number of cluster you can skip this step and use:* **`PlotProfilesClust`** *function*.
```{r,message=FALSE,warning=FALSE,eval=FALSE}
ClustProfiles <- ClustProfiles(prts, scaling = TRUE)
names(ClustProfiles)
```
   

```{r,message=FALSE, warning=FALSE,eval=FALSE}
# table of the profile and the recommended k that were found by the function 
head(ClustProfiles$optimalK)

# Table of clusters profile for each gene
head(ClustProfiles$ClusteredProfilesTable) 
```

## A graphic visualization of a profile  
The user choose the profile and the number of clusters to show, you can do that either after runing `ClustProfiles`, or not.  
Sometimes it takes too long to calculate all the clusters for all the profiles, so one can make clusters for a specific profile of his  choice, and also decide whether to supply the k or let the function calculate the optimal k for the selected profile.  
```{r,message=FALSE,warning=FALSE}

profilePlot <- PlotProfilesClust(prts,idx = "110000" , scaling = TRUE)
names(profilePlot)
```



***Genes in '110000' profile and their clusters (k was chosen by the function):***  
*Number of clusters (k) for '110000' is:* **`length(profilePlot $graphs)`**
```{r,message=FALSE,warning=FALSE,echo=FALSE}
length(profilePlot $graphs)
```

**Table of genes in '110000' profile, seperated to clusters:**
`head(profilePlot[[1]])`

```{r,message=FALSE,warning=FALSE,results="asis",echo=FALSE}

panderOptions("table.style","rmarkdown")
pander(head(profilePlot[[1]]))
```

***Line graphs of the genes expression pattern in '110000' profile, separated to clusters:***

```{r,message=FALSE,warning=FALSE,fig.width=10}
profilePlot$graphs
```
