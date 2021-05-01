[Página anterior<<](transcriptomic.md)  [Indice del Curso](Indice.md)

```bashd
## Carpeta del Bloque II

![](https://drive.google.com/drive/folders/1xgFHh4d-KqTPpymmqObihTz0pUewwahz?usp=sharing)

### Workflow RNA-seq

#trimming
for file in raw_data/*168*; do fastp -i $file -l 140 -q 30 -o trimmed/${file##*/}; done

for file in raw_data/*rep*; do fastp -i $file -l 70 -q 30 -o trimmed/${file##*/}; done
```
```bash
##mapping
STAR --runMode genomeGenerate --genomeDir index --genomeFastaFiles index/KY030782.fna --genomeSAindexNbases 7

for file in trimmed/*.fastq.gz; do STAR --genomeDir index --readFilesCommand gunzip -c --readFilesIn $file --outFileNamePrefix Star_out/${file##*/} --outSAMtype BAM Unsorted SortedByCoordinate --runThreadN 24; done

```
```bash
#assambly
for file in *.sortedByCoord.out.bam; do stringtie -p 16 -G ../index/KY030782.gtf -o ${file%Aligned.sortedByCoord.out.bam}.gtf -l ${file%Aligned.sortedByCoord.out.bam} $file; done

cat *.gtf > merge.txt

```

```bash
stringtie --merge -p 24 -G ../index/KY030782.gtf -o stringtie_merged.gtf merge.txt

for file in *.sortedByCoord.out.bam; do stringtie $file -e -B -p 24 -G stringtie_merged.gtf -o ballgown/${file%Aligned.sortedByCoord.out.bam}/${file%Aligned.sortedByCoord.out.bam}.gtf; done

cd ballgown


../../prepDE.py3
```
### Directorio de trabajo

``` bash
── RNA_Seq/
  │   └── genome/                    <- genoma de referencia (.FASTA) , anotación de genoma (.GTF/.GFF)
  │  
  │   └── reads/                     <- RNAseq data
  │  
  │   └── results/                   <- Archivos generados
  │       ├── quality/               <- Archivos de calidad
  │           ├──multiQC/            <- calidad conjunta
  │           ├──rawdata/            <- calidad de data cruda
  │       ├── trimmed/               <- Archivos filtrados
  │       ├── sortmerna/             <- Archivos filtrados de rRNA
  │           ├── aligned/           <- Secuencias alineadas a rRNA databases (con contenido de rRNA)
  │           ├── filtered/          <- Secuencias con rRNA removidos  (libre de rRNA)
  │           ├── logs/              <- logs
  │       ├── map/                   <- Alineamientos al genoma de referencia
  │           ├── aligned_bam/       <- Archivos de alineamiento (.BAM)
  │           ├── aligned_logs/      <- logs
  │       ├── counts/                <- Conteo de secuencias finales
  │  
  │   └── sortmerna_db/              <- rRNA databases
  │       ├── rRNA_databases/        <- rRNA (bacteria, archea y eukaryotes)
  │  
  │   └── scripts/                   <- Scripts usados con el curso


```r

##input matriz  de conteo
countData <- as.matrix(read.csv('gene_count_matrix.csv',row.names = 'gene_id'))
colData <- read.csv('phenodata.csv', row.names = 1)
all(rownames(colData) %in% colnames(countData))
countData <- countData[, rownames(colData)]
all(rownames(colData)==colnames(countData))

##generando colores por experimento (condicion)
col.condition <- c('noPep_5min'='green', 'pep_5min'='green', 'nopep_10min'= 'orange', 'pep_10min'= 'orange', 'nopep_20min'='red', 'pep_20min'='red')
colData$color <- col.condition[as.vector(colData$condition)]

##removiendo conteos nulos
prop.null <- apply(countData, 2, function(x) 100*mean(x==0))
barplot(prop.null,
        horiz=TRUE, cex.names=1, las=1,
        col=colData$color, xlab='% de conteos nulos')
countData <- countData[rowSums(countData) > 0,]

```

```r
##Analisis de expresión diferencial diferencial
library(DESeq2)
dds <- DESeqDataSetFromMatrix(countData = countData, colData = colData, design = ~condition)
dds <- DESeq(dds)

#removiendo genes con muy bajo conteo
dds <- dds[rowSums(counts(dds)) > 0,]
```
```r
##normalizando
dds.norm <- estimateSizeFactors(dds)

##varianza con respecto a la media
##calculando media y varianza
norm.counts <- counts(dds.norm, normalized=TRUE)
mean.counts <- rowMeans(norm.counts)
variance.counts <- apply(norm.counts, 1, var)

#generar resultados
dds.disp <- estimateDispersions(dds.norm)
alpha <- 0.05
lfcTH <- 1 
wald.test <- nbinomWaldTest(dds.disp)
res1 <- results(wald.test, contrast = c('condition','nopep_10min', 'pep_10min'), pAdjustMethod="BH")
res2 <- results(wald.test, contrast = c('condition','nopep_20min', 'pep_20min'), pAdjustMethod="BH")
summary(res1)
```
```r
##exportar resultados
write.csv(res1, file = 'nopep_vs_pep_10min.csv')
write.csv(res2, file = 'nopep_vs_pep_20min.csv')

#PCA
plotPCA(rlog)
rld <- rlog(wald.test, blind = FALSE)
View(rld)

#PCA
plotPCA(rlog)
```

```r
##VolcanoPlot
res1Volcano <- results(wald.test, contrast = c('condition','nopep_10min', 'pep_10min'))
keyvals1 <- rep('black', nrow(res1Volcano))
names(keyvals1) <- rep('Mid', nrow(res1Volcano))
keyvals1[which(res1Volcano$log2FoldChange > 1)] <- 'forestgreen'
names(keyvals1)[which(res1Volcano$log2FoldChange >1 )] <- 'high'
keyvals1[which(res1Volcano$log2FoldChange < -1)] <- 'red'
names(keyvals1)[which(res1Volcano$log2FoldChange < -1)] <- 'low'

library(EnhancedVolcano)
EnhancedVolcano(res1Volcano,
                lab = rownames(res1Volcano),
                x = 'log2FoldChange',
                y = 'pvalue',
                selectLab = rownames(res1Volcano)[which(names(keyvals1) %in% c('high', 'low'))],
                xlim = c(-6,6),
                xlab = bquote(~Log[2]~'FC'),
                pCutoff = 0.05,
                FCcutoff = 1.0,
                cutoffLineCol = 'blue',
                cutoffLineType = 'dashed',
                pointSize = 3.5,
                labSize = 4.5,
                shape = c(42,42,42,19),
                col = c('grey9','grey9','grey9','grey9'),
                colCustom = keyvals1,
                colAlpha = 1,
                legendPosition = 'top',
                legendLabSize = 15,
                legendIconSize = 5.0,
                border = 'partial',
                borderWidth = 1.5,
                borderColour = 'black')
```

```r
#heatmap
rld <- rlog(wald.test, blind = FALSE)
head(assay(rld), 3)
sampleDists <- dist(t(assay(rld)))
library("pheatmap")
library("RColorBrewer")
sampleDistMatrix <- as.matrix( sampleDists )
rownames(sampleDistMatrix) <- paste( rld$condition, sep = " - " )
colnames(sampleDistMatrix) <- NULL
colors <- colorRampPalette( rev(brewer.pal(9, "Blues")) )(255)
pheatmap(sampleDistMatrix,
         clustering_distance_rows = sampleDists,
         clustering_distance_cols = sampleDists,
         col = colors)
```

```r
library("genefilter")
topVarGenes <- head(order(rowVars(assay(rld)), decreasing = TRUE))
mat  <- assay(rld)[ topVarGenes, ]
mat  <- mat - rowMeans(mat)
anno <- as.data.frame(colData(rld))
rownames(anno) <- colnames(countData)
library("pheatmap")
#probablemente bote error, pero con la correcion anterior no deberia
pheatmap(mat, annotation_col = anno)
```

```r
#diagrama de venn
library(limma)
venn_data <- data.frame(p10min = res1$log2FoldChange >= 1 | res1$log2FoldChange <= -1,
                        p20min = res2$log2FoldChange >= 1 | res2$log2FoldChange <= -1)
vennDiagram(venn_data)

#PCA
plotPCA(rld)
```
