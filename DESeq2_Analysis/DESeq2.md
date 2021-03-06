---
title: "drap_oases_plant6_Davis_salmon"
output: 
  html_document: 
    keep_md: yes
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

#To do DE genes analysis just for Davis's libraries based on http://dibsi-rnaseq.readthedocs.io/en/latest/DE.html
#run on jetstream server

```{r}
setwd(dir = "/vol_c/quant/dammit_drap_oases/salmon_out/salmon_out_Davis/")
library(DESeq2)
library("lattice")
library(tximport)
library(readr)
library(gplots)
library(RColorBrewer)
source('~/plotPCAWithSampleNames.R')
```

#just for Davis's libraries

```{r}
dir<-"/vol_c/quant/dammit_drap_oases/salmon_out"
files_list = list.files()
files <- file.path(dir, "salmon_out_Davis",files_list, "quant.sf")
names(files) <- c("DS6","DF6","DL6","DR3","DS3","DF3","DL3","DR2","DS2","DF2","DL2","DR6")
files
print(file.exists(files))
```

#Grab the gene name and transcript ID file we generated by dammit

```{r}
gene_names <- read.csv("../Drap_Oases_Plant6_gene_name_id_dammit_namemap.csv3", header = F, sep = "\t")
dim(gene_names) #54128 2 I prefer just run for this annotation and for each Philipp's db see seperetaley. we can see whether they are availble in DE genes list.
cols<-c("transcript_id","gene_id")
colnames(gene_names)<-cols
#remove UniRef90
#gene_names$transcript_id <- gsub("UniRef90_", "", gene_names$transcript_id)
head(gene_names)
tx2gene<-gene_names
head(tx2gene)

#swap column
tx2gene[c(1,2)] <- tx2gene[c(2,1)]
head(tx2gene)
txi.salmon <- tximport(files, type = "salmon", tx2gene = tx2gene)
head(txi.salmon$counts)
colnames(txi.salmon$counts)
#change the order of colnames
txi.salmon$counts = txi.salmon$counts[,c("DR2", "DR3" , "DR6", "DS2","DS3","DS6","DF2","DF3","DF6","DL2","DL3","DL6")]
dim(txi.salmon$counts) #34684 12
save(txi.salmon,file="/vol_c/quant/dammit_drap_oases/salmon_out/salmon_out_Davis/txi.salmon.Rdata")
```
#Assign experimental variables

```{r}
condition = factor(c("R", "R", "R", "S", "S", "S", "F", "F", "F", "L", "L", "L"))
genotype = factor(c("2", "3", "6", "2","3","6","2","3","6","2","3","6"))

ExpDesign <- data.frame(row.names=colnames(txi.salmon$counts), condition = condition, genotype )
ExpDesign
```

#Run DESeq2

```{r}
dds <- DESeqDataSetFromTximport(txi.salmon, ExpDesign, ~condition+genotype)
dds <- DESeq(dds, betaPrior=FALSE)
save(dds,file="/vol_c/quant/dammit_drap_oases/salmon_out/salmon_out_Davis/dds_salmon.Rdata")
```

#Get counts

```{r}
counts_table = counts(dds, normalized=TRUE )
dim(counts_table)
#34684    12
#Filter out low expression transcripts:
filtered_norm_counts<-counts_table[!rowSums(counts_table==0)>=1, ]
filtered_norm_counts<-as.data.frame(filtered_norm_counts)
GeneID<-rownames(filtered_norm_counts)
filtered_norm_counts<-cbind(filtered_norm_counts,GeneID)
dim(filtered_norm_counts) #9829 13
head(filtered_norm_counts)
save(filtered_norm_counts,file="/vol_c/quant/dammit_drap_oases/salmon_out/salmon_out_Davis/filtered_norm_counts_salmon.Rdata")
#Estimate dispersion
pdf(file = "DispEsts.pdf", width = 12, height = 12);
plotDispEsts(dds)
dev.off()

#PCA
log_dds<-rlog(dds)
pdf(file = "PCA.pdf", width = 12, height = 12);
plotPCAWithSampleNames(log_dds, intgroup="condition", ntop=40000)
dev.off()
```

#Get DE results

```{r}
#log2 fold change, condition F vs R 
res<-results(dds,contrast=c("condition","F","R"))
head(res)
res_ordered<-res[order(res$padj),]
GeneID<-rownames(res_ordered)
res_ordered<-as.data.frame(res_ordered)
res_genes<-cbind(res_ordered,GeneID)
dim(res_genes) #34684 7 
head(res_genes)
res_genes_merged <- merge(res_genes,filtered_norm_counts,by=unique("GeneID"))
dim(res_genes_merged) # 9829 19
head(res_genes_merged)
res_ordered<-res_genes_merged[order(res_genes_merged$padj),]
#get arid of NA in pvalue and padj
#x <- is.na(res_ordered$padj)
#write.table(x,"x.txt",sep="\t")
res_ordered <- res_ordered[-c(9805:9829), ]
dim(res_ordered) #9804 19 after removing rownames with pvalue or padj NA
write.csv(res_ordered, file="/vol_c/quant/dammit_drap_oases/salmon_out/salmon_out_Davis/drap_oases_plant6_salmon_DESeq_all.csv" )
```

#Set a threshold cutoff of padj<0.05 and ± log2FC 1

```{r}
resSig = res_ordered[res_ordered$padj < 0.05, ] #change this to use pvalue or padj < 0.1 or pvalue <0.05 for now put it as a default

dim(resSig) #505 19
resSig = resSig[resSig$log2FoldChange > 1 | resSig$log2FoldChange < -1,]
dim(resSig) #505 19
write.csv(resSig,file="/vol_c/quant/dammit_drap_oases/salmon_out/salmon_out_Davis/drap_oases_plant6_DESeq_padj0.05_log2FC1.csv")
```

#MA plot with gene names

```{r}
pdf(file = "MA_drap_oasess_FvsR.pdf", width = 50, height = 40)
plot(log2(res_ordered$baseMean), res_ordered$log2FoldChange, col=ifelse(res_ordered$padj < 0.05, "red","gray67"),main="drap_oases (padj<0.05, log2FC = ±1)",xlim=c(1,20),pch=20,cex=1,ylim=c(-12,12))
abline(h=c(-1,1), col="blue")
genes<-resSig$GeneID
mygenes <- resSig[,]
baseMean_mygenes <- mygenes[,"baseMean"]
log2FoldChange_mygenes <- mygenes[,"log2FoldChange"]
text(log2(baseMean_mygenes),log2FoldChange_mygenes,labels=genes,pos=2,cex=0.60)
dev.off()
```

#Heatmap

```{r}
d<-resSig
dim(d) #505 19 
head(d)
colnames(d)
d<-d[,c(8:19)]
d<-as.matrix(d)
d<-as.data.frame(d)
d<-as.matrix(d)
rownames(d) <- resSig[,1]
head(d)
dim(d) #505 12 these genes are DE genes F vs R in 12 libraries
```

```{r}
pdf(file = "Heatmap_drap_oasess_FvsR.pdf", height=15,width=8)
jpeg('Heatmap_drap_oasess_FvsR.jpg' )
#par(mar=c(4, 0, 2, 2)) #bottom, left, top, right
hr <- hclust(as.dist(1-cor(t(d), method="pearson")), method="complete")
mycl <- cutree(hr, h=max(hr$height/1.5))
clusterCols <- rainbow(length(unique(mycl)))
myClusterSideBar <- clusterCols[mycl]
myheatcol <- greenred(75)
heatmap.2(d, main="Drap_oases (padj<0.05, log2FC = ±1)", 
          Rowv=as.dendrogram(hr),
          #cexRow=0.75,cexCol=0.8,
          srtCol= 90,
          adjCol = c(NA,0),offsetCol=2.5, 
          Colv=NA, dendrogram="row", 
          scale="row", col=myheatcol, 
          density.info="none", 
          trace="none", RowSideColors= myClusterSideBar)
dev.off()

```

#save the txt file encluding gene ID with 6 cluster, I want to use this file to find go term for uniref ID to see the go enrichment

```{r}
write.table(mycl,"/vol_c/quant/dammit_drap_oases/salmon_out/salmon_out_Davis/mycl.txt",sep="\t") #used this file to see the number of cluster for each gene ID
```

#compare Stem vs Root

```{r}
#log2 fold change, condition S vs R 
resSR<-results(dds,contrast=c("condition","S","R"))
head(resSR)
resSR_ordered<-resSR[order(resSR$padj),]
GeneID<-rownames(resSR_ordered)
resSR_ordered<-as.data.frame(resSR_ordered)
resSR_genes<-cbind(resSR_ordered,GeneID)
dim(resSR_genes) #34684 7 
head(resSR_genes)
resSR_genes_merged <- merge(resSR_genes,filtered_norm_counts,by=unique("GeneID"))
dim(resSR_genes_merged) # 9829 19
head(resSR_genes_merged)
resSR_ordered<-resSR_genes_merged[order(resSR_genes_merged$padj),]
#get arid of NA in pvalue and padj
#x <- is.na(resSR_ordered$padj)
#write.table(x,"x.txt",sep="\t")
resSR_ordered <- resSR_ordered[-c(9783:9829), ]
dim(resSR_ordered) #9782   19 after removing rownames with pvalue or padj NA
write.csv(resSR_ordered, file="drap_oases_plant6_DESeq_SR_all.csv" )
```

#Set a threshold cutoff of padj<0.05 and ± log2FC 1

```{r}
resSigSR = resSR_ordered[resSR_ordered$padj < 0.05, ] #change this to use pvalue or padj < 0.1 or pvalue <0.05 for now put it as a default

dim(resSigSR) #140 19
resSigSR = resSigSR[resSigSR$log2FoldChange > 1 | resSigSR$log2FoldChange < -1,]
dim(resSigSR) #140 19
write.csv(resSigSR,file="drap_oases_plant6_DESeq_padj0.05_log2FC1_SR.csv")
```

#MA plot with gene names

```{r}
par(30,20)
pdf(file = "MA_drap_oasess_SvsR.pdf", width = 30, height = 20)
jpeg('MA_drap_oasess_SvsR.jpg' )
plot(log2(resSR_ordered$baseMean), resSR_ordered$log2FoldChange, col=ifelse(resSR_ordered$padj < 0.05, "red","gray67"),main="drap_oases (padj<0.05, log2FC = ±1)",xlim=c(1,20),pch=20,cex=1,ylim=c(-12,12))
abline(h=c(-1,1), col="blue")
genes<-resSigSR$GeneID
mygenes <- resSigSR[,]
baseMean_mygenes <- mygenes[,"baseMean"]
log2FoldChange_mygenes <- mygenes[,"log2FoldChange"]
text(log2(baseMean_mygenes),log2FoldChange_mygenes,labels=genes,pos=2,cex=0.60)
dev.off()
```

#Heatmap

```{r}
d<-resSigSR
dim(d) #140 
head(d)
colnames(d)
d<-d[,c(8:19)]
d<-as.matrix(d)
d<-as.data.frame(d)
d<-as.matrix(d)
rownames(d) <- resSigSR[,1]
head(d)
dim(d) #140 12 these genes are DE genes F vs R in 12 libraries
```

```{r}
#par("mar")
#5.1 4.1 4.1 2.1
pdf("Heatmap_drap_oasess_SvsR.pdf",height=15,width=8)
par(mar=c(4, 0, 2, 13)) #bottom, left, top, right
#jpeg('Heatmap_drap_oasess_SvsR.jpg', height = 10, width = 8)
hr <- hclust(as.dist(1-cor(t(d), method="pearson")), method="complete")
mycl <- cutree(hr, h=max(hr$height/1.5))
clusterCols <- rainbow(length(unique(mycl)))
myClusterSideBar <- clusterCols[mycl]
myheatcol <- greenred(75)
heatmap.2(d, main="Drap_oases (padj<0.05, log2FC = ±1)", 
          Rowv=as.dendrogram(hr),
          #cexRow=0.75,cexCol=0.8,
          srtCol= 90,
          adjCol = c(NA,0),offsetCol=2.5, 
          Colv=NA, dendrogram="row", 
          scale="row", col=myheatcol, 
          density.info="none", 
          trace="none", RowSideColors= myClusterSideBar)

dev.off() 

```

#save the txt file encluding gene ID with 3 cluster, I want to use this file to find go term for uniref ID to see the go enrichment

```{r}
write.table(mycl,"mycl.SvsR.txt",sep="\t") #used this file to see the number of cluster for each gene ID
```

#compare Leaf vs Root

```{r}
#log2 fold change, condition L vs R 
resLR<-results(dds,contrast=c("condition","L","R"))
head(resLR)
resLR_ordered<-resLR[order(resLR$padj),]
GeneID<-rownames(resLR_ordered)
resLR_ordered<-as.data.frame(resLR_ordered)
resLR_genes<-cbind(resLR_ordered,GeneID)
dim(resLR_genes) #34684 7 
head(resLR_genes)
resLR_genes_merged <- merge(resLR_genes,filtered_norm_counts,by=unique("GeneID"))
dim(resLR_genes_merged) # 9829 19
head(resLR_genes_merged)
resLR_ordered<-resLR_genes_merged[order(resLR_genes_merged$padj),]
#get arid of NA in pvalue and padj
#x <- is.na(resLR_ordered$padj)
#write.table(x,"x.txt",sep="\t")
resLR_ordered <- resLR_ordered[-c(9829:9829), ]
dim(resLR_ordered) #9828  19 after removing rownames with pvalue or padj NA
write.csv(resLR_ordered, file="drap_oases_plant6_DESeq_LR_all.csv" )
```

#Set a threshold cutoff of padj<0.05 and ± log2FC 1

```{r}
resSigLR = resLR_ordered[resLR_ordered$padj < 0.05, ] #change this to use pvalue or padj < 0.1 or pvalue <0.05 for now put it as a default

dim(resSigLR) #1155 19
resSigLR = resSigLR[resSigLR$log2FoldChange > 1 | resSigLR$log2FoldChange < -1,]
dim(resSigLR) #1155 19
write.csv(resSigLR,file="drap_oases_plant6_DESeq_padj0.05_log2FC1_LR.csv")
```

#MA plot with gene names

```{r}
pdf(file = "MA_drap_oasess_LvsR.pdf", width = 30, height = 20)
jpeg('MA_drap_oasess_LvsR.jpg' )
plot(log2(resLR_ordered$baseMean), resLR_ordered$log2FoldChange, col=ifelse(resLR_ordered$padj < 0.05, "red","gray67"),main="drap_oases (padj<0.05, log2FC = ±1)",xlim=c(1,20),pch=20,cex=1,ylim=c(-12,12))
abline(h=c(-1,1), col="blue")
genes<-resSigLR$GeneID
mygenes <- resSigLR[,]
baseMean_mygenes <- mygenes[,"baseMean"]
log2FoldChange_mygenes <- mygenes[,"log2FoldChange"]
text(log2(baseMean_mygenes),log2FoldChange_mygenes,labels=genes,pos=2,cex=0.60)
dev.off()
```

#Heatmap

```{r}
d<-resSigLR
dim(d) #1155
head(d)
colnames(d)
d<-d[,c(8:19)]
d<-as.matrix(d)
d<-as.data.frame(d)
d<-as.matrix(d)
rownames(d) <- resSigLR[,1]
head(d)
dim(d) #1155 12 these genes are DE genes L vs R in 18 libraries
```

```{r}
#par("mar")
#5.1 4.1 4.1 2.1
pdf("Heatmap_drap_oasess_LvsR.pdf",height=10,width=8)
par(mar=c(4, 0, 2, 13)) #bottom, left, top, right
#jpeg('Heatmap_drap_oasess_LvsR.jpg')
hr <- hclust(as.dist(1-cor(t(d), method="pearson")), method="complete")
mycl <- cutree(hr, h=max(hr$height/1.5))
clusterCols <- rainbow(length(unique(mycl)))
myClusterSideBar <- clusterCols[mycl]
myheatcol <- greenred(75)
heatmap.2(d, main="Drap_oases (padj<0.05, log2FC = ±1)", 
          Rowv=as.dendrogram(hr),
          #cexRow=0.75,cexCol=0.8,
          srtCol= 90,
          adjCol = c(NA,0),offsetCol=2.5, 
          Colv=NA, dendrogram="row", 
          scale="row", col=myheatcol, 
          density.info="none", 
          trace="none", RowSideColors= myClusterSideBar)

dev.off() 

```

#save the txt file encluding gene ID with 8 cluster, I want to use this file to find go term for uniref ID to see the go enrichment

```{r}
write.table(mycl,"mycl.LvsR.txt",sep="\t") #used this file to see the number of cluster for each gene ID
```

#compare Flower vs Stem

```{r}
#log2 fold change, condition F vs S
resFS<-results(dds,contrast=c("condition","F","S"))
head(resFS)
resFS_ordered<-resFS[order(resFS$padj),]
GeneID<-rownames(resFS_ordered)
resFS_ordered<-as.data.frame(resFS_ordered)
resFS_genes<-cbind(resFS_ordered,GeneID)
dim(resFS_genes) #34684 7 
head(resFS_genes)
resFS_genes_merged <- merge(resFS_genes,filtered_norm_counts,by=unique("GeneID"))
dim(resFS_genes_merged) # 9829  19
head(resFS_genes_merged)
resFS_ordered<-resFS_genes_merged[order(resFS_genes_merged$padj),]
#get arid of NA in pvalue and padj
#x <- is.na(resFS_ordered$padj)
#write.table(x,"x.txt",sep="\t")
dim(resFS_ordered) #9829  19 after removing rownames with pvalue or padj NA
write.csv(resFS_ordered, file="drap_oases_plant6_DESeq_FS_all.csv" )
```

#Set a threshold cutoff of padj<0.05 and ± log2FC 1

```{r}
resSigFS = resFS_ordered[resFS_ordered$padj < 0.05, ] #change this to use pvalue or padj < 0.1 or pvalue <0.05 for now put it as a default

dim(resSigFS) #215  19
resSigFS = resSigFS[resSigFS$log2FoldChange > 1 | resSigFS$log2FoldChange < -1,]
dim(resSigFS) #215  19
write.csv(resSigFS,file="drap_oases_plant6_DESeq_padj0.05_log2FC1_FS.csv")
```

#MA plot with gene names

```{r}
pdf(file = "MA_drap_oasess_FvsS.pdf", width = 30, height = 20)
jpeg('MA_drap_oasess_FvsS.jpg' )
plot(log2(resFS_ordered$baseMean), resFS_ordered$log2FoldChange, col=ifelse(resFS_ordered$padj < 0.05, "red","gray67"),main="drap_oases (padj<0.05, log2FC = ±1)",xlim=c(1,20),pch=20,cex=1,ylim=c(-12,12))
abline(h=c(-1,1), col="blue")
genes<-resSigFS$GeneID
mygenes <- resSigFS[,]
baseMean_mygenes <- mygenes[,"baseMean"]
log2FoldChange_mygenes <- mygenes[,"log2FoldChange"]
text(log2(baseMean_mygenes),log2FoldChange_mygenes,labels=genes,pos=2,cex=0.60)
dev.off()
```

#Heatmap

```{r}
d<-resSigFS
dim(d) #215
head(d)
colnames(d)
d<-d[,c(8:19)]
d<-as.matrix(d)
d<-as.data.frame(d)
d<-as.matrix(d)
rownames(d) <- resSigFS[,1]
head(d)
dim(d) #215 12 these genes are DE genes F vs S in 12 libraries
```

```{r}
#par("mar")
#5.1 4.1 4.1 2.1
pdf("Heatmap_drap_oasess_FvsS.pdf",height=10,width=8)
par(mar=c(4, 0, 2, 13)) #bottom, left, top, right
#jpeg('Heatmap_drap_oasess_FvsS.jpg')
hr <- hclust(as.dist(1-cor(t(d), method="pearson")), method="complete")
mycl <- cutree(hr, h=max(hr$height/1.5))
clusterCols <- rainbow(length(unique(mycl)))
myClusterSideBar <- clusterCols[mycl]
myheatcol <- greenred(75)
heatmap.2(d, main="Drap_oases (padj<0.05, log2FC = ±1)", 
          Rowv=as.dendrogram(hr),
          #cexRow=0.75,cexCol=0.8,
          srtCol= 90,
          adjCol = c(NA,0),offsetCol=2.5, 
          Colv=NA, dendrogram="row", 
          scale="row", col=myheatcol, 
          density.info="none", 
          trace="none", RowSideColors= myClusterSideBar)

dev.off() 

```

#save the txt file encluding gene ID with 4 cluster, I want to use this file to find go term for uniref ID to see the go enrichment

```{r}
write.table(mycl,"mycl.FvsS.txt",sep="\t") #used this file to see the number of cluster for each gene ID
```

#compare Flower vs Leaf

```{r}
#log2 fold change, condition F vs L
resFL<-results(dds,contrast=c("condition","F","L"))
head(resFL)
resFL_ordered<-resFL[order(resFL$padj),]
GeneID<-rownames(resFL_ordered)
resFL_ordered<-as.data.frame(resFL_ordered)
resFL_genes<-cbind(resFL_ordered,GeneID)
dim(resFL_genes) #34684 7 
head(resFL_genes)
resFL_genes_merged <- merge(resFL_genes,filtered_norm_counts,by=unique("GeneID"))
dim(resFL_genes_merged) # 9829   19
head(resFL_genes_merged)
resFL_ordered<-resFL_genes_merged[order(resFL_genes_merged$padj),]
#get arid of NA in pvalue and padj
#x <- is.na(resFL_ordered$padj)
#write.table(x,"x.txt",sep="\t")
resFL_ordered <- resFL_ordered[-c(9827:9829), ]
dim(resFL_ordered) #9826   19 after removing rownames with pvalue or padj NA
write.csv(resFL_ordered, file="drap_oases_plant6_DESeq_FL_all.csv" )
```

#Set a threshold cutoff of padj<0.05 and ± log2FC 1

```{r}
resSigFL = resFL_ordered[resFL_ordered$padj < 0.05, ] #change this to use pvalue or padj < 0.1 or pvalue <0.05 for now put it as a default

dim(resSigFL) #341  19
resSigFL = resSigFL[resSigFL$log2FoldChange > 1 | resSigFL$log2FoldChange < -1,]
dim(resSigFL) #341  19
write.csv(resSigFL,file="drap_oases_plant6_DESeq_padj0.05_log2FC1_FL.csv")
```

#MA plot with gene names

```{r}
pdf(file = "MA_drap_oasess_FvsL.pdf", width = 30, height = 20)
jpeg('MA_drap_oasess_FvsL.jpg' )
plot(log2(resFL_ordered$baseMean), resFL_ordered$log2FoldChange, col=ifelse(resFL_ordered$padj < 0.05, "red","gray67"),main="drap_oases (padj<0.05, log2FC = ±1)",xlim=c(1,20),pch=20,cex=1,ylim=c(-12,12))
abline(h=c(-1,1), col="blue")
genes<-resSigFL$GeneID
mygenes <- resSigFL[,]
baseMean_mygenes <- mygenes[,"baseMean"]
log2FoldChange_mygenes <- mygenes[,"log2FoldChange"]
text(log2(baseMean_mygenes),log2FoldChange_mygenes,labels=genes,pos=2,cex=0.60)
dev.off()
```

#Heatmap

```{r}
d<-resSigFL
dim(d) #341
head(d)
colnames(d)
d<-d[,c(8:19)]
d<-as.matrix(d)
d<-as.data.frame(d)
d<-as.matrix(d)
rownames(d) <- resSigFL[,1]
head(d)
dim(d) #341 12 these genes are DE genes F vs L in 12 libraries
```

```{r}
#par("mar")
#5.1 4.1 4.1 2.1
pdf("Heatmap_drap_oasess_FvsL.pdf",height=10,width=8)
par(mar=c(4, 0, 2, 13)) #bottom, left, top, right
#jpeg('Heatmap_drap_oasess_FvsL.jpg')
hr <- hclust(as.dist(1-cor(t(d), method="pearson")), method="complete")
mycl <- cutree(hr, h=max(hr$height/1.5))
clusterCols <- rainbow(length(unique(mycl)))
myClusterSideBar <- clusterCols[mycl]
myheatcol <- greenred(75)
heatmap.2(d, main="Drap_oases (padj<0.05, log2FC = ±1)", 
          Rowv=as.dendrogram(hr),
          #cexRow=0.75,cexCol=0.8,
          srtCol= 90,
          adjCol = c(NA,0),offsetCol=2.5, 
          Colv=NA, dendrogram="row", 
          scale="row", col=myheatcol, 
          density.info="none", 
          trace="none", RowSideColors= myClusterSideBar)

dev.off() 

```

#save the txt file encluding gene ID with 10 cluster, I want to use this file to find go term for uniref ID to see the go enrichment

```{r}
write.table(mycl,"mycl.FvsL.txt",sep="\t") #used this file to see the number of cluster for each gene ID
```

#compare Stem vs Leaf

```{r}
#log2 fold change, condition S vs L
resSL<-results(dds,contrast=c("condition","S","L"))
head(resSL)
resSL_ordered<-resSL[order(resSL$padj),]
GeneID<-rownames(resSL_ordered)
resSL_ordered<-as.data.frame(resSL_ordered)
resSL_genes<-cbind(resSL_ordered,GeneID)
dim(resSL_genes) #34684 7 
head(resSL_genes)
resSL_genes_merged <- merge(resSL_genes,filtered_norm_counts,by=unique("GeneID"))
dim(resSL_genes_merged) # 9829   19
head(resSL_genes_merged)
resSL_ordered<-resSL_genes_merged[order(resSL_genes_merged$padj),]
#get arid of NA in pvalue and padj
#x <- is.na(resSL_ordered$padj)
#write.table(x,"x.txt",sep="\t")
resSL_ordered <- resSL_ordered[-c(9404:9829), ]
dim(resSL_ordered) #9403  19 after removing rownames with pvalue or padj NA
write.csv(resSL_ordered, file="drap_oases_plant6_DESeq_SL_all.csv" )
```

#Set a threshold cutoff of padj<0.05 and ± log2FC 1

```{r}
resSigSL = resSL_ordered[resSL_ordered$padj < 0.05, ] #change this to use pvalue or padj < 0.1 or pvalue <0.05 for now put it as a default

dim(resSigSL) #16 19
resSigSL = resSigSL[resSigSL$log2FoldChange > 1 | resSigSL$log2FoldChange < -1,]
dim(resSigSL) #16 19
write.csv(resSigSL,file="drap_oases_plant6_DESeq_padj0.05_log2FC1_SL.csv")
```

#MA plot with gene names

```{r}
pdf(file = "MA_drap_oasess_SvsL.pdf", width = 30, height = 20)
jpeg('MA_drap_oasess_SvsL.jpg' )
plot(log2(resSL_ordered$baseMean), resSL_ordered$log2FoldChange, col=ifelse(resSL_ordered$padj < 0.05, "red","gray67"),main="drap_oases (padj<0.05, log2FC = ±1)",xlim=c(1,20),pch=20,cex=1,ylim=c(-12,12))
abline(h=c(-1,1), col="blue")
genes<-resSigSL$GeneID
mygenes <- resSigSL[,]
baseMean_mygenes <- mygenes[,"baseMean"]
log2FoldChange_mygenes <- mygenes[,"log2FoldChange"]
text(log2(baseMean_mygenes),log2FoldChange_mygenes,labels=genes,pos=2,cex=0.60)
dev.off()
```

#Heatmap

```{r}
d<-resSigSL
dim(d) #16
head(d)
colnames(d)
d<-d[,c(8:19)]
d<-as.matrix(d)
d<-as.data.frame(d)
d<-as.matrix(d)
rownames(d) <- resSigSL[,1]
head(d)
dim(d) #16 12 these genes are DE genes S vs L in 12 libraries
```

```{r}
#par("mar")
#5.1 4.1 4.1 2.1
pdf("Heatmap_drap_oasess_SvsL.pdf",height=10,width=8)
par(mar=c(4, 0, 2, 13)) #bottom, left, top, right
#jpeg('Heatmap_drap_oasess_SvsL.jpg')
hr <- hclust(as.dist(1-cor(t(d), method="pearson")), method="complete")
mycl <- cutree(hr, h=max(hr$height/1.5))
clusterCols <- rainbow(length(unique(mycl)))
myClusterSideBar <- clusterCols[mycl]
myheatcol <- greenred(75)
heatmap.2(d, main="Drap_oases (padj<0.05, log2FC = ±1)", 
          Rowv=as.dendrogram(hr),
          #cexRow=0.75,cexCol=0.8,
          srtCol= 90,
          adjCol = c(NA,0),offsetCol=2.5, 
          Colv=NA, dendrogram="row", 
          scale="row", col=myheatcol, 
          density.info="none", 
          trace="none", RowSideColors= myClusterSideBar)

dev.off() 

```
#grep contig name for uniref ID

```{r}
#awk '{print $1}' mycl.txt | sed 's/"//g' > mycl.txt2 
#wc -l mycl.txt2
#nano mycl.txt2 remove x
#grep -F -f mycl.txt2 ../Drap_Oases_Plant6_gene_name_id_dammit_namemap.csv3 > filtered.FvsR.tsv
#wc filtered.FvsR.tsv 
#less filtered.FvsR.tsv 
#cat filtered.FvsR.tsv | awk '{print $1}' filtered.FvsR.tsv | sort | uniq | wc -l
#cat filtered.FvsR.tsv | awk '{print $1}' filtered.FvsR.tsv | sort | uniq > filtered.FvsR.sort.tsv
#check the final file
#awk '{print $1}' filtered.FvsR.sort.tsv | sort | uniq | > temp1
#awk '{print $1}' mycl.txt2 | sort | uniq >temp2
#comm temp1 temp2
#wc -l filtered.FvsR.sort.tsv
#cat mycl.txt | awk '{if ($2==1) print;}' > mycl.txt.cluster1 run for each cluster
#cat mycl.txt.cluster1 | sed 's/"//g' | awk '{print $1}' > mycl.txt.cluster1.v2
#grep -F -f mycl.txt.cluster1.v2 ../Drap_Oases_Plant6_gene_name_id_dammit_namemap.csv3 > filtered.FvsR.cluster1.tsv
#cat filtered.FvsR.cluster1.tsv | awk '{print $2}' > filtered.FvsR.cluster1.name.tsv
#I do not think above command is right for getting gene ID becuase for one UniRef got multiple gene ID so it is better to grep Uniref to see in DAVID with gene ontology red "significantly"
#DAVID start analysis uniprot_accession Gene Accession Conversion Tool,submit converted list,go ontology,pick red one, click on chart

#cat mycl.txt.cluster1 | awk '{print $1}' | sed 's/"//g' | head -500 | sed 's/UniRef90_//g' | wc
# 170     170    1802
```

#save the txt file encluding gene ID with 4 cluster, I want to use this file to find go term for uniref ID to see the go enrichment

```{r}
write.table(mycl,"mycl.SvsL.txt",sep="\t") #used this file to see the number of cluster for each gene ID
```

