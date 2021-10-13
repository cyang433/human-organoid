# human-organoid-bulk
## Software Requirements
![image](https://github.com/cyang433/human-organoid/blob/main/hmtp.pdf)
The analysis is based on R 4.0. 
Users should install the following packages prior to using the code:
```R
install.packages(c('rdist','SCORPIUS','Rmagic','pheatmap','Linnorm','reshape2','reticulate','plyr','RColorBrewer','stringr','limma','ManiNetCluster','ComplexHeatmap','circlize'))
# import ManiNetCluster form "https://github.com/daifengwanglab/ManiNetCluster"
```
Besides,bulk cell data of human and organoid and some functions for processing data are also needed for our project.
```R
source('../src/func.r')
load('bulk_start.RData')
```
## demo

### Step 0: data preprocessing
```R
rdata1 = rdata_human; meta1=meta_human
rdata2 = rdata_organoid; meta2=meta_organoid
# create a data set that only contains human information
regions = unique(meta1$Region)
form.data1=rdata1[,meta1$Species=='Human' & meta1$Region %in% regions]
form.meta1=meta1[meta1$Species=='Human' & meta1$Region %in% regions,]
form.data2=rdata2
form.meta2=meta2
# add a column representing the observation time to each meta data set.
form.meta1$time=form.meta1$Period   
form.meta2$time=form.meta2$Days
```

### Step 1: Find different expression genes at any time
Use the limma package to perform differential analysis at each time to find differentially expressed genes.
```R
DE.list<-function(data,meta) {
        genes = c()
        for (t in unique(meta$time)) {
                condition=as.factor(meta$time==t)
                design = model.matrix(~0+condition)  # Create a design matrix containing time information             
                fit=lmFit(data,design)  # Linear model fitting
                fit = eBayes(fit)  # Bayesian test  
                top_table = topTable(fit,number=10000)  # Get the differentially expressed gene name
                genes = c(genes,row.names(top_table[top_table$adj.P.Val<0.05,]))  # combine into a list
                 }
        genes = unique(genes)
        return(genes)
}
deg_list1 = DE.list(form.data1,form.meta1)
deg_list2 = DE.list(form.data2,form.meta2)
```
### Step 2: Focus on expression of interest
The genes we are interested in are obtained by taking the intersection of genes differentially expressed in human and organoid cells.
```R
sel.genes = intersect(intersect(deg_list1,deg_list2),unique(all.rec$gene))  # Select genes of interest
sel.data1 = form.data1[row.names(form.data1) %in% sel.genes,]  # Select expression based genes
sel.data1 = sel.data1[!duplicated(row.names(sel.data1)),] 
sel.meta1=form.meta1  #on human data
sel.data2 = form.data2[row.names(form.data2) %in% sel.genes,] 
sel.data2 = sel.data2[!duplicated(row.names(sel.data2)),] 
sel.meta2=form.meta2  #on organoid data
```
```R
# perform logarithmic transformation 
exp1 = sel.data1[order(row.names(sel.data1)),order(sel.meta1$time)];
sel.meta1=sel.meta1[order(sel.meta1$time),]
exp2 = sel.data2[order(row.names(sel.data2)),order(sel.meta2$time)];
sel.meta2=sel.meta2[order(sel.meta2$time),]
# reordering processing
ps.mat1 = t(exp1);ps.time1=sel.meta1$time
ps.mat2 = t(exp2);ps.time2=sel.meta2$time
```
### Step 3: Alignment
ManiNetCluster employs manifold learning to uncover and match local and non-linear structures among networks, and identifies cross-network functional links.We use ManiNetCluster simultaneously aligns and clusters co-expression.
```R
algn_res = runMSMA_dtw(ps.mat1,ps.mat2)
df2 = algn_res[[3]]
df2$time = c(sel.meta1$time,sel.meta2$time)  # Add the time information to the aligned matrix
```
```R
# calculate pairwise distances between cells after MSMA
pair_dist = apply(df2[df2$data=='sample1',c(3:5)],1,function(x) {
        d = apply(df2[df2$data=='sample2',c(3:5)],1,function(y) eu.dist(x,y))
})

row.names(pair_dist)=row.names(ps.mat2)
colnames(pair_dist)=row.names(ps.mat1)
sim_dist = 1/(1+pair_dist)

# hmtp3-2nd align
cols1 = c(brewer.pal(9,'YlGnBu')[c(2:9)],brewer.pal(11,'BrBG')[c(8:11)]);names(cols1) = unique(ps.time1)
annot1=rowAnnotation(H_time=factor(ps.time1,levels=unique(ps.time1)),col=list(H_time=cols1))
cols2 = c(brewer.pal(9,'YlOrRd')[c(2:9)],brewer.pal(11,'BrBG')[rev(1:4)]);names(cols2) = unique(ps.time2)
annot2=columnAnnotation(O_time=factor(ps.time2,levels=unique(ps.time2)),col=list(O_time=cols2))

sim_mat = t(sim_dist)
pdf('hmtp.pdf')
Heatmap(sim_mat,name='human_vs_humanOrg',show_row_names=F,show_column_names=F,cluster_rows=F,cluster_columns=F,
left_annotation=annot1,top_annotation=annot2,col=colorRamp2(c(0.8,max(sim_dist)),c('white','red')))
dev.off()
```
### Step 4: Visualization
Use the aligned matrix to draw 3D scatter plots for human and organoid,visualize the time information of each point in color.
```R
# plot 3D trajectors
pdf('3D1.pdf')
time.cols1 = colorRampPalette(brewer.pal(n=9,'Greens'))(12)
#time.cols1 = c(brewer.pal(9,'YlGnBu')[c(2:9)],brewer.pal(11,'BrBG')[c(8:11)])
res = data.frame(df2[df2$data=='sample1',])
res0 = data.frame(df2)
library(plot3D)
s3d<-scatter3D(x=res[,3],y=res[,4],z=res[,5],colvar=as.numeric(mapvalues(res$time,names(table(res$time)),c(2:13))),col = c(time.cols1),pch=c(16,17)[as.numeric(as.factor(res$data))],colkey=F,theta = 300, phi = 30,cex=2,
xlim=c(min(res0$Val0),max(res0$Val0)),
ylim=c(min(res0$Val1),max(res0$Val1)),
zlim=c(min(res0$Val2),max(res0$Val2)))
legend("top", legend = levels(as.factor(res$data)), pch = c(16, 17),inset = -0.1, xpd = TRUE, horiz = TRUE)
legend("right", legend = levels(as.factor(res$time)), col = c(time.cols1),pch=16,inset =-0.05, xpd = TRUE, horiz = F,cex=1.2)
dev.off()

#time.cols2 = c(brewer.pal(9,'YlOrRd')[c(2:9)],brewer.pal(11,'BrBG')[rev(1:4)])
#c25 <- c( "dodgerblue2", "#E31A1C",  "green4", "#6A3D9A",  "#FF7F00",  "black", "gold1", "skyblue2", "#FB9A99",  "palegreen2", "#CAB2D6",  "#FDBF6F", "gray70", "khaki2", "maroon", "orchid1", "deeppink1", "blue1", "steelblue4", "darkturquoise", "green1", "yellow4", "yellow3", "darkorange4", "brown")
#time.cols2 = c25[1:12]

#time.cols2 = colorRampPalette(brewer.pal(9, "YlOrBr"))(12) #for human
time.cols2 = colorRampPalette(brewer.pal(9, "Blues"))(12) #for organoid

res = data.frame(df2[df2$data=='sample2',])
#res = data.frame(df2)
res0 = data.frame(df2)
library(plot3D)
pdf('3D2.pdf',width=10)
s3d<-scatter3D(x=res[,3],y=res[,4],z=res[,5],pch=24,col='gray',bg=time.cols2[as.numeric(mapvalues(res$time,names(table(res$time)),c(1:12)))],colkey=F,theta = 300, phi = 30,cex=2,lwd=1,
xlim=c(min(res0$Val0),max(res0$Val0)),
ylim=c(min(res0$Val1),max(res0$Val1)),
zlim=c(min(res0$Val2),max(res0$Val2)))
legend("top", legend = levels(as.factor(res$data)), pch = c(16, 17),inset = -0.1, xpd = TRUE, horiz = TRUE)

legend("right", legend = levels(as.factor(res$time)), col = c(time.cols2),pch=16,inset = -0.05, xpd = TRUE, horiz = F,cex=1.2)
dev.off()
```
Finally,draw the corrplot on timepoint wise averaged similarity.
```R
library(corrplot)
sim_avg = matrix(0,nrow=length(unique(ps.time1)),ncol=length(unique(ps.time2)))

i = 0
for(t1 in unique(ps.time1)) {
        i = i+1
        j = 0
        for (t2 in unique(ps.time2)) {
                j = j+1
                sim_tmp = sim_mat[ps.time1==t1,ps.time2==t2]
                avg_tmp = mean(sim_tmp)
                sim_avg[i,j] = avg_tmp
        }
}

sim_avg = t(sim_avg)
colnames(sim_avg) = unique(ps.time1)
row.names(sim_avg) = unique(ps.time2)

library(RColorBrewer)
pdf('corr.plot.bulk.human.vs.org1.pdf')
corrplot(sim_avg,col=rev(colorRampPalette(brewer.pal(9,'PRGn'))(200)),cl.lim=c(0.7,1),is.corr=F,tl.col="black")
dev.off()
```



