## Introduction

In this workflow, we will create protein-protein interaction (PPI)
networks for both biopsy locations ileum and rectum. Then these networks
will be extended with pathways from WikiPathways database to create
PPI-pathway networks. Finally, MCL (Markov Clustering) network
clustering algorithm will be applied to get clusters within the network.

## Setup

Installing and loading required libraries

## Importing dataset

The data will be read for the disease on two biopsy locations

``` r
##Obtain all Differentially Expressed Gene data from step 3:
setwd('..')
work_DIR <- getwd()

#read all DEG data
CD.ileum <- read.delim("4-pathway_analysis/output/DEGs_CD_ileum",sep = "\t", header = TRUE)
CD.rectum <- read.delim("4-pathway_analysis/output/DEGs_CD_rectum", sep = "\t",header = TRUE)
UC.ileum <- read.delim("4-pathway_analysis/output/DEGs_UC_ileum",sep = "\t", header = TRUE)
UC.rectum <- read.delim("4-pathway_analysis/output/DEGs_UC_rectum", sep = "\t",header = TRUE)

# Set Working Directory back to current folder
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
work_DIR <- getwd()

#Listing all up and down regulated genes separately for CD:
CD.up.ileum   <-unique(CD.ileum[CD.ileum$log2FC_ileum > 0.58,])
colnames(CD.up.ileum) <- c ("HGNC_symbol", "ENTREZ", "log2FC_CD", "pvalue_CD")
CD.down.ileum <-unique(CD.ileum[CD.ileum$log2FC_ileum < -0.58,])
colnames(CD.down.ileum) <- c ("HGNC_symbol", "ENTREZ", "log2FC_CD", "pvalue_CD")
CD.up.rectum   <-unique(CD.rectum[CD.rectum$log2FC_rectum > 0.58,])
colnames(CD.up.rectum) <- c ("HGNC_symbol", "ENTREZ", "log2FC_CD", "pvalue_CD")
CD.down.rectum <-unique(CD.rectum[CD.rectum$log2FC_rectum < -0.58,])
colnames(CD.down.rectum) <- c ("HGNC_symbol", "ENTREZ", "log2FC_CD", "pvalue_CD")
#Listing all up and down regulated genes separately for UC:
UC.up.ileum   <-unique(UC.ileum[UC.ileum$log2FC_ileum > 0.58,])
colnames(UC.up.ileum) <- c ("HGNC_symbol", "ENTREZ", "log2FC_UC", "pvalue_UC")
UC.down.ileum <-unique(UC.ileum[UC.ileum$log2FC_ileum < -0.58,])
colnames(UC.down.ileum) <- c ("HGNC_symbol", "ENTREZ", "log2FC_UC", "pvalue_UC")
UC.up.rectum   <-unique(UC.rectum[UC.rectum$log2FC_rectum > 0.58,])
colnames(UC.up.rectum) <- c ("HGNC_symbol", "ENTREZ", "log2FC_UC", "pvalue_UC")
UC.down.rectum <-unique(UC.rectum[UC.rectum$log2FC_rectum < -0.58,])
colnames(UC.down.rectum) <- c ("HGNC_symbol", "ENTREZ", "log2FC_UC", "pvalue_UC")
```

## Finding overlapping genes between diseases on ileum biopsy location

``` r
######################################FOR ILEUM biopsy location#######################################
# overlap genes between CD down and UC down
merged.ileum.downCDdownUC <- merge(x=CD.down.ileum, y=UC.down.ileum, by=c('ENTREZ', 'HGNC_symbol'), all.x=FALSE, all.y=FALSE)

# overlap genes between CD up and UC down
merged.ileum.upCDdownUC <- merge(x=CD.up.ileum,y=UC.down.ileum, by=c('ENTREZ', 'HGNC_symbol'),all.x=FALSE, all.y=FALSE)

# overlap genes between CD up and UC up
merged.ileum.upCDupUC <- merge(x=CD.up.ileum,y=UC.up.ileum,by=c('ENTREZ', 'HGNC_symbol'), all.x=FALSE, all.y=FALSE)

# overlap genes between CD down and UC up
merged.ileum.downCDupUC <- merge(x=CD.down.ileum,y=UC.up.ileum,by=c('ENTREZ', 'HGNC_symbol'),all.x=FALSE, all.y=FALSE)

#merge all DEG with corresponding logFC for both diseases
DEG.overlapped_ileum <- rbind(merged.ileum.downCDdownUC, merged.ileum.upCDdownUC, merged.ileum.upCDupUC, merged.ileum.downCDupUC)
if(!dir.exists("output")) dir.create("output")
write.table(DEG.overlapped_ileum ,"output/DEG.overlapped_ileum",row.names=FALSE,col.names = TRUE,quote= FALSE, sep = "\t")
##############################################################################################
```

## Finding overlapped genes between diseases on rectum biopsy location

``` r
######################################FOR RECTUM biopsy location#######################################
# overlap genes between CD down and UC down
merged.rectum.downCDdownUC <- merge(x=CD.down.rectum,y=UC.down.rectum,by=c('ENTREZ', 'HGNC_symbol'),all.x=FALSE, all.y=FALSE)

# overlap genes between CD up and UC down
merged.rectum.upCDdownUC <- merge(x=CD.up.rectum,y=UC.down.rectum,by=c('ENTREZ', 'HGNC_symbol'),all.x=FALSE, all.y=FALSE)

# overlap genes between CD up and UC up
merged.rectum.upCDupUC <- merge(x=CD.up.rectum,y=UC.up.rectum,by=c('ENTREZ', 'HGNC_symbol'),all.x=FALSE, all.y=FALSE)

# overlap genes between CD down and UC up
merged.rectum.downCDupUC <- merge(x=CD.down.rectum,y=UC.up.rectum,by=c('ENTREZ', 'HGNC_symbol'),all.x=FALSE, all.y=FALSE)

#merge all DEG with corresponding logFC for both diseases
DEG.overlapped_rectum <- rbind(merged.rectum.downCDdownUC, merged.rectum.upCDdownUC, merged.rectum.upCDupUC, merged.rectum.downCDupUC)
write.table(DEG.overlapped_rectum ,"output/DEG.overlapped_rectum",row.names=FALSE,col.names = TRUE,quote= FALSE, sep = "\t")
##############################################################################################
```

## Create Protein-Protein Interaction (PPI) network for selected biopsy location

``` r
##Remove data objects which are not needed for further processing:
rm(list=setdiff(ls(), c("DEG.overlapped_ileum", "DEG.overlapped_rectum")))

## Select a location to analyse (options; ileum or rectum)
location <- "ileum"

if (location == "ileum"){deg <- DEG.overlapped_ileum}else if(location == "rectum"){deg <- DEG.overlapped_rectum}else{print("Location not recognized")}

#check that cytoscape is connected
cytoscapePing()
#close session before starting --> this overwrites existing networks!
closeSession(FALSE)
networkName = paste0("PPI_network_",location)
#create a PPI network using overlapped DEGs between CD and UC
#first take the deg input list as a query
x <- readr::format_csv(as.data.frame(deg$ENTREZ), col_names=F, escape = "double", eol =",")
#then use the below function to convert a command string into a CyREST query URL, executes a GET request, 
#and parses the result content into an R list object. Same as commandsGET
commandsRun(paste0('string protein query cutoff=0.7', ' newNetName=',networkName, ' query=',x,' limit=0'))
```

    ## [1] "Loaded network 'STRING network - PPI_network_ileum' with 188 nodes and 32 edges"

``` r
#get proteins (nodes) from the constructed network: #Query term: entrez IDs, display name: HGNC symbols.
proteins <- RCy3::getTableColumns(columns=c("query term", "display name"))
#get edges from the network
ppis     <- RCy3::getTableColumns(table="edge", columns=c("name"))
#split extracted edge information into source-target format
ppis     <- data.frame(do.call('rbind', strsplit(as.character(ppis$name),' (pp) ',fixed=TRUE)))
#merge obtained nodes and edges to get entrez IDs for each source genes 
ppis.2   <- merge(ppis, proteins, by.x="X1", by.y="display name", all.x=T)
#change column names
colnames(ppis.2) <- c("s", "t", "source")
#merge again to add entrez IDs of target genes 
ppis.3   <- merge(ppis.2, proteins, by.x="t", by.y="display name", all.x=T)
colnames(ppis.3)[4] <-"target"
#ppi3 stores interaction between all proteins so add new column represeting type of interaction
ppis.3$interaction <- "PPI"
#add col names to protein
colnames(proteins) <- c("id","label")
proteins$type <- "protein"

###############get all pathways from WIKIPATHWAYS #################
##Work with local file (for publication), or new download:
work_DIR <- getwd()
pathway_data <- "local" #Options: local, new
if (pathway_data == "local") {
  wp.hs.gmt <-list.files(work_DIR, pattern="wikipathways", full.names=FALSE)
  paste0("Using local file, from: ", wp.hs.gmt )
}else if(pathway_data == "new"){ 
#below code should be performed first to handle the ssl certificate error while downloading pathways 
options(RCurlOptions = list(cainfo = paste0( tempdir() , "/cacert.pem" ), ssl.verifypeer = FALSE))
#downloading latest pathway gmt files for human 
wp.hs.gmt <- rWikiPathways::downloadPathwayArchive(organism="Homo sapiens", format = "gmt")
  paste0("Using new data, from: ", wp.hs.gmt)}else{print("Pathway data type not recognized")
  }
```

    ## [1] "Using local file, from: wikipathways-20220210-gmt-Homo_sapiens.gmt"

``` r
#all wp and gene information stored in wp2gene object
wp2gene   <- rWikiPathways::readPathwayGMT(wp.hs.gmt)
#filter out  pathways that does not consist of any differentially expressed genes 
wp2gene.filtered <- wp2gene [wp2gene$gene %in% deg$ENTREZ,]

#change column names 
colnames(wp2gene.filtered)[3] <- c("source")
colnames(wp2gene.filtered)[5] <- c("target")
#add new column for representing interaction type
wp2gene.filtered$interaction <- "Pathway-Gene"

#store only wp information 
pwy.filtered <- unique( wp2gene [wp2gene$gene %in% deg$ENTREZ,c(1,3)])
colnames(pwy.filtered) <- c("label", "id")
pwy.filtered$type <- "pathway"
colnames(pwy.filtered) <- c("label","id", "type")

#get genes 
genes <- unique(deg[,c(1,2), drop=FALSE])
genes$type <- "gene"
colnames(genes) <- c("id","label","type")
genes$id <- as.character(genes$id)
#genes and pathways are separate nodes and they need to be merged
nodes.ppi <- dplyr::bind_rows(genes,pwy.filtered)
rownames(nodes.ppi) <- NULL
edges.ppi <- unique(dplyr::bind_rows(ppis.3[,c(3,4,5)], wp2gene.filtered[,c(3,5,6)]))
rownames(edges.ppi) <- NULL
```

## Create Protein-Protein-pathway-interaction (PPPI) network for selected biopsy location

``` r
#create a network name 
networkName = paste0("PPI_Pathway_Network_",location)

###########Create PPI-pathway network###
RCy3::createNetworkFromDataFrames(nodes= nodes.ppi, edges = edges.ppi, title=networkName, collection=location)
```

    ## networkSUID 
    ##       12769

``` r
RCy3::loadTableData(nodes.ppi, data.key.column = "label", table="node", table.key.column = "label")
```

    ## [1] "Success: Data loaded in defaultnode table"

``` r
RCy3::loadTableData(deg, data.key.column = "ENTREZ", table.key.column = "id")
```

    ## [1] "Success: Data loaded in defaultnode table"

``` r
###########Visual style#################
RCy3::copyVisualStyle("default","ppi")#Create a new visual style (ppi) by copying a specified style (default)
RCy3::setNodeLabelMapping("label", style.name="ppi")
```

    ## NULL

``` r
RCy3::lockNodeDimensions(TRUE, style.name="ppi")#Set a boolean value to have node width and height fixed to a single size value.

#threshold is set based of differential expressed gene criteria
data.values<-c(-0.58,0,0.58) 
#red-blue color schema chosen
node.colors <- c(brewer.pal(length(data.values), "RdBu"))
#nodes are split to show both log2fc values for both diseases 
RCy3::setNodeCustomHeatMapChart(c("log2FC_CD","log2FC_UC"), slot = 2, style.name = "ppi", colors = c("#CC3300","#FFFFFF","#6699FF","#CCCCCC"))
RCy3::setVisualStyle("ppi")
```

    ##                 message 
    ## "Visual Style applied."

``` r
# Saving output
if(!dir.exists("output")) dir.create("output")
outputName = paste0 ("output/PPI_Pathway_Network_", wp.hs.gmt, location,".png")
png.file <- file.path(getwd(), outputName)
exportImage(png.file,'PNG', zoom = 500)
```

    ##                                                                                                                                                                                                    file 
    ## "/home/deniseslenter/Documents/GitHub/Transcriptomics_Metabolomics_Analysis/transcriptomics_analysis/6-network_analysis/output/PPI_Pathway_Network_wikipathways-20220210-gmt-Homo_sapiens.gmtileum.png"

## Clustering obtained networks

``` r
#we will continue with the same session used for pppi networks
#to check cytoscape is connected
cytoscapePing()

#Install the Clustermaker2 app (if not available already)
if("Clustermaker2" %in% commandsHelp("")) print("Success: the Clustermaker2 app is installed") else print("Warning: Clustermaker2 app is not installed. Please install the Clustermaker2 app before proceeding.")
```

    ## [1] "Available namespaces:"
    ## [1] "Warning: Clustermaker2 app is not installed. Please install the Clustermaker2 app before proceeding."

``` r
if(!"Clustermaker2" %in% commandsHelp("")){
  installApp("Clustermaker2")
}
```

    ## [1] "Available namespaces:"

``` r
networkName = paste0 ("PPI_Pathway_Network_",location)
#to get network name of the location 
networkSuid = getNetworkSuid(networkName)
setCurrentNetwork(network=networkSuid)
#create cluster command
clustermaker <- paste("cluster mcl createGroups=TRUE showUI=TRUE network=SUID:",networkSuid, sep="")
#run the command in cytoscape
res <- commandsGET(clustermaker)
#total number of clusters 
cat("Total number of clusters for",location, as.numeric(gsub("Clusters: ", "", res[1])))
```

    ## Total number of clusters for ileum 65

``` r
#change pathway node visualization
pathways <- RCy3::selectNodes(nodes="pathway", by.col = "type")
RCy3::setNodeColorBypass(node.names = pathways$nodes, "#D3D3D3")
RCy3::setNodeBorderWidthBypass(node.names = pathways$nodes, 10)

#export image
outputName = paste0 ("output/PPI_Pathway_Network_",location, wp.hs.gmt,"_clustered",".png")
png.file <- file.path(getwd(), outputName)
exportImage(png.file,'PNG', zoom = 500)
```

    ##                                                                                                                                                                                                              file 
    ## "/home/deniseslenter/Documents/GitHub/Transcriptomics_Metabolomics_Analysis/transcriptomics_analysis/6-network_analysis/output/PPI_Pathway_Network_ileumwikipathways-20220210-gmt-Homo_sapiens.gmt_clustered.png"

``` r
#save session
cys.file <- file.path(getwd(), "output/PPI_Pathway_Network_",wp.hs.gmt, location,"_clustered",".cys")
#saveSession(cys.file) 

#if the new data file exist, remove it (so it does not conflict with running the code against the local file) 
if(pathway_data == "new") file.remove(wp.hs.gmt)
```

##Print session info and remove large datasets:

``` r
##Print session info:
sessionInfo()
```

    ## R version 4.2.0 (2022-04-22)
    ## Platform: x86_64-pc-linux-gnu (64-bit)
    ## Running under: Ubuntu 18.04.6 LTS
    ## 
    ## Matrix products: default
    ## BLAS:   /usr/lib/x86_64-linux-gnu/blas/libblas.so.3.7.1
    ## LAPACK: /usr/lib/x86_64-linux-gnu/lapack/liblapack.so.3.7.1
    ## 
    ## locale:
    ##  [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
    ##  [3] LC_TIME=nl_NL.UTF-8        LC_COLLATE=en_US.UTF-8    
    ##  [5] LC_MONETARY=nl_NL.UTF-8    LC_MESSAGES=en_US.UTF-8   
    ##  [7] LC_PAPER=nl_NL.UTF-8       LC_NAME=C                 
    ##  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
    ## [11] LC_MEASUREMENT=nl_NL.UTF-8 LC_IDENTIFICATION=C       
    ## 
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ## 
    ## other attached packages:
    ## [1] RColorBrewer_1.1-3   rWikiPathways_1.16.0 RCy3_2.16.0         
    ## [4] readr_2.1.2          tidyr_1.2.0          dplyr_1.0.9         
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] pbdZMQ_0.3-7        tidyselect_1.1.2    xfun_0.31          
    ##  [4] repr_1.1.4          purrr_0.3.4         vctrs_0.4.1        
    ##  [7] generics_0.1.2      htmltools_0.5.2     stats4_4.2.0       
    ## [10] yaml_2.3.5          base64enc_0.1-3     utf8_1.2.2         
    ## [13] XML_3.99-0.9        rlang_1.0.2         pillar_1.7.0       
    ## [16] glue_1.6.2          DBI_1.1.2           bit64_4.0.5        
    ## [19] BiocGenerics_0.42.0 uuid_1.1-0          lifecycle_1.0.1    
    ## [22] stringr_1.4.0       evaluate_0.15       uchardet_1.1.0     
    ## [25] knitr_1.39          tzdb_0.3.0          fastmap_1.1.0      
    ## [28] parallel_4.2.0      curl_4.3.2          fansi_1.0.3        
    ## [31] IRdisplay_1.1       backports_1.4.1     BiocManager_1.30.17
    ## [34] IRkernel_1.3        vroom_1.5.7         graph_1.74.0       
    ## [37] jsonlite_1.8.0      bit_4.0.4           fs_1.5.2           
    ## [40] rjson_0.2.21        hms_1.1.1           digest_0.6.29      
    ## [43] stringi_1.7.6       RJSONIO_1.3-1.6     cli_3.3.0          
    ## [46] tools_4.2.0         bitops_1.0-7        magrittr_2.0.3     
    ## [49] base64url_1.4       RCurl_1.98-1.6      tibble_3.1.7       
    ## [52] crayon_1.5.1        pkgconfig_2.0.3     ellipsis_0.3.2     
    ## [55] data.table_1.14.2   assertthat_0.2.1    rmarkdown_2.14     
    ## [58] httr_1.4.3          rstudioapi_0.13     R6_2.5.1           
    ## [61] compiler_4.2.0

## Last, we create a Jupyter notebook from this script

``` r
#Jupyter Notebook file
if(!"devtools" %in% installed.packages()) BiocManager::install("devtools")
devtools::install_github("mkearney/rmd2jupyter", force=TRUE)
```

    ## 
    ## * checking for file ‘/tmp/RtmpY1AS9e/remotes5c662a3c5f4e/mkearney-rmd2jupyter-d2bd2aa/DESCRIPTION’ ... OK
    ## * preparing ‘rmd2jupyter’:
    ## * checking DESCRIPTION meta-information ... OK
    ## * checking for LF line-endings in source and make files and shell scripts
    ## * checking for empty or unneeded directories
    ## Omitted ‘LazyData’ from DESCRIPTION
    ## * building ‘rmd2jupyter_0.1.0.tar.gz’

``` r
library(devtools)
library(rmd2jupyter)
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
rmd2jupyter("Network_analysis.Rmd")
```
