# featurecounts

#### 设置工作路径 ####  
```r
readpath = '/home/jjyang/yjj/ATAC/blacklist_rm/'
path = '/home/jjyang/yjj/ATAC/'
```

## 1. 加载macs3输出的narrowPeak，并且过滤了blacklists  
```r
peak <- lapply(list.files(readpath, "*.narrowPeak"), function(x){
  return(readPeakFile(file.path(readpath, x)))
})

names(peak) <- str_split_fixed(list.files(readpath, "*.narrowPeak"), '_p', n = 2)[,1]
txdb <- TxDb.Mmusculus.UCSC.mm10.knownGene

#### 注释  
peakAnnoList <- lapply(peak, annotatePeak, TxDb=txdb, tssRegion=c(-3000, 3000), 
                       annoDb="org.Mm.eg.db", verbose=FALSE, overlap="all")
peakAnno_df <- lapply(peakAnnoList, function(x){x <- as.data.frame(x)})

peakAnno_df <- lapply(peakAnno_df, function(x){
  colnames(x)[6:12] <- c('name', 'score', 'strand', 'signalValue', 'log10pValue', 'log10qValue', 'summit_peak_start')
  return(x)
})
```

## 2.基础绘图，peak占比，TSS热图等   
```r
plotDistToTSS(peakAnnoList)
ggplot2::ggsave(paste0(path, "results/plotDistToTSS.pdf"),
                height = 5, width = 8, dpi = 300, limitsize = FALSE)

plotAnnoBar(peakAnnoList)
ggplot2::ggsave(paste0(path, "results/plotAnnoBar.pdf"),
                height = 5, width = 8, dpi = 300, limitsize = FALSE)

promoter <- getPromoters(TxDb=txdb, upstream=3000, downstream=3000)

tagMatrixList <- lapply(peak, getTagMatrix, windows=promoter)

plotAvgProf(tagMatrixList, xlim =c(-3000, 3000), conf=0.95, resample=500, facet="row")

ggplot2::ggsave(paste0(path, "results/plotAvgProf.pdf"),
                height = 20, width = 8, dpi = 300, limitsize = FALSE)
```

## 3.筛选saf文件所需要的列（GeneID, Chr, Start, End and Strand）
```r
region_bed <- lapply(peakAnno_df, function(x){
  x <- x[, c("SYMBOL","seqnames","start","end", "strand")]
  x$SYMBOL <- paste0('Peak', 1:nrow(x),'_', x$SYMBOL)
  colnames(x) <- c("GeneID","Chr","Start","End","Strand")
  return(x)
})

pm_bed <- lapply(peakAnno_df, function(x){
  x <- x[-grep("Rik$", ignore.case = F, x$SYMBOL),]
  x <- x[grep("Promoter", ignore.case = F, x$annotation), ]
  x <- x[, c("SYMBOL","seqnames","start","end", "strand")]
  x$SYMBOL <- paste0('Peak', 1:nrow(x),'_', x$SYMBOL)
  colnames(x) <- c("GeneID","Chr","Start","End","Strand")
  return(x)
})

gb_bed <- lapply(peakAnno_df, function(x){
  x <- x[-grep("Rik$", ignore.case = F, x$SYMBOL),]
  x <- x[grep("Promoter|Distal Intergenic", ignore.case = F, x$annotation), ]
  x <- x[, c("SYMBOL","seqnames","start","end", "strand")]
  x$SYMBOL <- paste0('Peak', 1:nrow(x),'_', x$SYMBOL)
  colnames(x) <- c("GeneID","Chr","Start","End","Strand")
  return(x)
})
```

储存在本地count文件夹内，需要提前创建该文件夹
```r

save(peakAnno_df, peakAnnoList, region_bed, pm_bed, gb_bed, file = paste0(path, 'count/Anno_df.RData'))

#### 输出saf文件格式的bed
for (i in 1:length(pm_bed)) {
  write.table(x = pm_bed[[i]],
              file = paste0(path, 'count/', names(pm_bed)[i], '_pm.bed'),
              sep = "\t", row.names = FALSE, col.names = colnames(pm_bed[[i]]), quote = FALSE)
}

for (i in 1:length(gb_bed)) {
  write.table(x = gb_bed[[i]],
              file = paste0(path, 'count/', names(gb_bed)[i], '_gb.bed'),
              sep = "\t", row.names = FALSE, col.names = colnames(gb_bed[[i]]), quote = FALSE)
}
```
---

## 4.进行count计算  

制作bam文件和saf文件的路径，要求这两个数据框是一一对应的   
```r
bamPath <- paste0(path, 'bam/')
bamNames <- dir(bamPath, pattern = "last.bam$") 
bamPath <- sapply(bamNames, function(x){paste0(bamPath,x)})     
bamPath <- data.frame(bamPath); rownames(bamPath) <- str_split_fixed(bamNames, "_", n = 2)[,1]

safPath <- paste0(path, 'count/')
safNames <- dir(safPath, pattern = "_pm.bed$") 
safPath <- sapply(safNames, function(x){paste0(safPath,x)})   
safPath <- data.frame(safPath)

bamPath
safPath
```
featureCounts计算  
具体参考  
- https://subread.sourceforge.net/SubreadUsersGuide.pdf

> `countMultiMappingReads` A multi-mapping read is a read that maps to more than one location in the reference genome. There are multiple options for counting such reads. Users can specify the ‘-M’ option (set `countMultiMappingReads` to TRUE in R) to fully count every alignment reported for a multimapping read (each alignment carries 1 count), or specify both ‘-M’ and ‘–fraction’ options (set both `countMultiMappingReads` and `fraction` to TRUE in R) to count each alignment fractionally (each alignment carries 1/x count where x is the total number of alignments reported for the read), or do not count such reads at all (this is the default behavior in SourceForge Subread package; In R, you need to set `countMultiMappingReads` to FALSE).

> `allowMultiOverlap` A multi-overlapping read is a read that overlaps more than one meta-feature when counting reads at meta-feature level or overlaps more than one feature when counting reads at feature level. The decision of whether or not to counting these reads is often determined by the experiment type. We recommend that reads or fragments overlapping more than one gene are not counted for RNA-seq experiments, because any single fragment must originate from only one of the target genes but the identity of the true target gene cannot be confidently determined. On the other hand, we recommend that multi-overlapping reads or fragments are counted for ChIP-seq experiments because for example epigenetic modifications inferred from these reads may regulate the biological functions of all their overlapping genes.

> `allowMultiOverlap` By default, featureCounts does not count multi-overlapping reads. Users can specify the ‘-O’ option (set `allowMultiOverlap` to TRUE in R) to fully count them for each overlapping metafeature/feature (each overlapping meta-feature/feature receives a count of 1 from a read), or specify both ‘-O’ and ‘–fraction’ options (set both `allowMultiOverlap` and `fraction` to TRUE in R) to assign a fractional count to each overlapping meta-feature/feature (each overlapping meta-feature/feature receives a count of 1/y from a read where y is the total number of meta-features/features overlapping with the read).

```r
peak_counts <- list()
for (i in 1:nrow(safPath)) {
  peak_counts[[i]] <- Rsubread::featureCounts(files = as.character(bamPath[i,]),
                                              allowMultiOverlap = T, 
                                              fraction = T,
                                              countMultiMappingReads = T,
                                              countChimericFragments = F,
                                              nthreads = 8,
                                              isPairedEnd = T,
                                              annot.ext = as.character(safPath[i,]))
  rm(i)
}
```


    pkc <- function(name){
      bamPath <- paste0("chip-nt-rawdata/bam/", his)
      bamNames <- dir(bamPath, pattern = ".bam$") 
      bamPath <- sapply(bamNames, function(x){paste(bamPath, x, sep='/')})   
      bamPath <- data.frame(bamPath); rownames(bamPath) <- str_split_fixed(bamNames, "_", n = 2)[,1]
      #
      safPath <- paste0("chip-nt-rawdata/", his, "/" , name, "_saf")
      safNames <- dir(safPath, pattern = ".saf$") 
      safPath <- sapply(safNames, function(x){paste(safPath, x, sep='/')})   
      safPath <- data.frame(safPath) 

      peak_counts <- list()
      for (i in 1:5) {
        peak_counts[[i]] <- Rsubread::featureCounts(files = as.character(bamPath[c(2*i-1, 2*i),]),
                                                    allowMultiOverlap = T,
                                                    countMultiMappingReads = F,
                                                    countChimericFragments = F,
                                                    nthreads = 8,
                                                    isPairedEnd = T,
                                                    annot.ext = as.character(safPath[i,]))
        rm(i)
      }

      count_list <- lapply(peak_counts, function(x){
        x <- x$counts
        x <- data.frame(x)
        x$SYMBOL <- rownames(x)
        return(x)
      })

      names(count_list) <- c('e11.5', 'e12.5', 'e13.5', 'e14.5', 'e15.5')
      raw_counts <- Reduce(merge, count_list)

      colnames(raw_counts)[-1] <- c('nt_e11.5_1', 'nt_e11.5_2', 
                                    'nt_e12.5_1', 'nt_e12.5_2', 
                                    'nt_e13.5_1', 'nt_e13.5_2', 
                                    'nt_e14.5_1', 'nt_e14.5_2', 
                                    'nt_e15.5_1', 'nt_e15.5_2')

      save(count_list, file = paste0("chip-nt-rawdata/", his, "/", his, "_", name, "_count_list.RData"))
      write.csv(raw_counts, paste0("chip-nt-rawdata/", his, "/", his, "_", name, "_counts.csv"), row.names = F)
    }
    his <- 'H3K9me3'
    pkc('pm')
    pkc('gb')
    pkc('dis')
