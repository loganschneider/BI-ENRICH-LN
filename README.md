# BI-ENRICH-LN

The code here is an reproducible example of BI-ENRICH pathway analysis that was applied for figure 2 in the paper, Meta-analysis of over 100,000 individuals of European ancestry identifies 19 genome-wide significant risk loci for restless legs syndrome. 

## hardware

| item | configuration|
|---|---:|
|CPU| 16 cores, 32 threads; 2 x Xeon E5-2630 v3 (2.4GHz); |
|RAM| 128G ; 8 x 16G Samsung DIMM (2133 MHz); |

## load data

```r
load("data/annotation_matrix_gs.rda") # annotation_matrix_gs
colnames(annotation_matrix_gs)
```
A matrix indicate annotation of 16903 genes in 6121 gene sets.

```r
load("data/candidategeneset.rda") # annotation_matrix_gs
head(cs) # two columns: gene name; loci id
```

## BI-ENRICH pathway enrichment

### calculate z score 

```r
ncores = 12;               # number of cores for parallel computing
bienrich_pval = 0.05;      # p-value cutoff to perform greedy search for the potention multiple mixed gene sets.
shrinking_geneset = TRUE ; # remove large and redundant gene sets to accelerate computing 

source("src/calculate_geneset_zscore.R")

zcs2gs_res_list <- calculate_geneset_zscore( cs=cs,
				             annotation_matrix_gs = annotation_matrix_gs, 
					     ncores=ncores, 
					     bienrich_pval=bienrich_pval,
				             shrinking_geneset=TRUE)

zcs2gs_res <- zcs2gs_res_list[[1]]
annotation_matrix <- zcs2gs_res_list[[2]]
save(annotation_matrix,file="annotation_matrix_gs_sk.rda")
head(zcs2gs_res[order(zcs2gs_res[,1],decreasing=T),],3)
```

|           |z|                   anno|
|---|:---:|---:|	   
104 |5.689770 |GO_LOCOMOTORY_BEHAVIOR|
115 |5.566328|            GO_BEHAVIOR|
14  |5.480649 |    UNIKB_Neurogenesis|

### calculate random sampling z-score

```r
permute_gs <- calculate_geneset_zscore_sampling(times=1000,cs,annotation_matrix, ncores=8, bienrich_pval=0.05,shrinking_geneset=FALSE)
save(permute_gs,file="permute_gs.1000.Rda")
# the computing will take few hours; you can load data/permute_gs.1000.Rda for the next step;
load("data/permute_gs.1000.Rda") 
```

### calculate random sampling adjusted significance

```r
gs_enrich_stats <- calculate_geneset_sampling_adjusted(zcs2gs_res,permute_gs)
head(gs_enrich_stats[order(gs_enrich_stats$zraw,decreasing=T),],3)
```

|Geneset|       pvalue(sampling)|   fdr|     zraw|                        genes|
|---|:---:|:---:|:---:|---:|
|GO_LOCOMOTORY_BEHAVIOR |3.0363e-07 |0.000 |5.689770 |      BTBD9 CLN6 HOXB8 MEIS1
|          GO_BEHAVIOR |1.2200e-04 |0.007 |5.566328 | BTBD9 CLN6 DACH1 HOXB8 MEIS1
|  UNIKB_Neurogenesis |2.6919e-06 |0.000 |5.480649 |     MDGA1 MYT1 NTNG1 SEMA6D



### calculate null-GWAS permutation adjusted significance

```r
# null-GWAS phenotype generated by PLINK --make-perm-pheno
# calculate z-score for the genes in the top 20 signficant loci of each null-GWAS runs
# 1000 files for top 20 signficant loci of each run in data/perm_null_GWAS/perm_null_GWAS.top20genes.tar
# example files see: ALL_HWE5_Info05.HCT005.GENO020.dbSNP146.*.clumped.top20.gene
# run the following code, please tar xf data/perm_null_GWAS/perm_null_GWAS.top20genes.tar

## parLapply loop to calculate z-score for each of the 1000 null-GWAS runs
##   better to use cluster to accelerate the 1000 GWAS test and z-score calculation; 

library(parallel)
cl = makeCluster(rep('localhost',2)) ; # a simple example for a SMP linux machine
zcs2gs_res_list <- parLapply(cl,1:1000,function(simi,...){
                                cs = read.table(
			                     paste("data/perm_null_GWAS/ALL_HWE5_Info05.HCT005.GENO020.dbSNP146.",
			                            simi+2,
						    ".clumped.top20.gene",sep=""),
						    sep="\t",header = F,stringsAsFactors = F)[,2:1] ; 
			        source("src/calculate_geneset_zscore.R")
			        parcalculate_geneset_zscore( cs=cs,
				             annotation_matrix_gs = annotation_matrix_gs, 
					     ncores=ncores, 
					     bienrich_pval=bienrich_pval,
				             shrinking_geneset=FALSE)
				save(zcs2tsres,file=paste("res.gs.",simi,".Rda",sep="")) ; 
				zcs2tsres},
		              annotation_matrix_gs = annotation_matrix, 
			      ncores=ncores, 
			      bienrich_pval=bienrich_pval)

rdas <- list.files(pattern="res.gs.*.Rda")
d <- t(sapply(rdas,function(x){
  #print(x)
  load(x);
  as.numeric(zcs2tsres[,1])
}))
rownames(d) <- zcs2tsres[,2]

empvalue <- sapply(1:nrow(d),function(x){
  (sum(d[x,]>zcs2gs_res[x,1])+1)/1000
})

gsemp <- data.frame(geneset=zcs2gs_res[,2],pvalue=empvalue,fdr=p.adjust(empvalue,"fdr"), rawz = zcs2gs_res[,1])
head(gsemp[order(gsemp[,4]),])[1:3,]
```



|                      |               geneset  |pvalue(perm)|   fdr|     z|
|---|:---:|:---:|:---:|---:|
|GO_LOCOMOTORY_BEHAVIOR| GO_LOCOMOTORY_BEHAVIOR  |0.002 |0.029| 5.689770|
|GO_DNA_REPAIR         |          GO_DNA_REPAIR  |0.005 |0.036| 4.978007|
|UNIKB_Neurogenesis    |     UNIKB_Neurogenesis  |0.005 |0.036| 5.480649|

