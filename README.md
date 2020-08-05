# He lab m6A-seq_analysis_workflow
A pipeline to process m6A-seq data and down stream analysis for he lab members. 

## Prepare data for analysis
### 0. Design your experiment carefully
Data analysis would benefit greatly from careful design of your experiment. Be sure each batch of m6A-Seq contains a normal control. Use the same analysis pipeline for all your datasets when you need to compare them.

### 1. Download the data from the genomic core. 
You can use [filezilla](https://filezilla-project.org/download.php?type=client) to download the data from che@osrfftp.uchicago.edu (port: 21) and upload it to **youraccount**@128.135.225.178 (port: 22).  
Alternatively, you can ssh log into **youraccount**@128.135.225.81 and go to the directory by `cd /directory of your favorite` where you want to analyze your data. Then use the 
```
sftp -r che@osrfftp.uchicago.edu:/data_file_or_directory_name ./
## for example
sftp -r che@osrfftp.uchicago.edu:/170823_NS500373_0154_AHG3MTBGX3-CHe-JN-63S-pl1 ./
```
to directly download the data from sequencing facility server to the lab analysis server. 


### 2. Organize and rename files for processing
In order to be easily processed, rename the files to short names without space and postfix `.IN.fastq.gz` or `.m6A.fastq.gz` for input and IP respectively.  
For example, 
```
Ctl1.IN.fastq.gz
Ctl1.m6A.fastq.gz
...
FTO_KO5.IN.fastq.gz
FTO_KO5.m6A.fastq.gz
```
### 3. Backup your data to our backup server
Preparing a detailed descriptive metadata table is highly recommended. Then backup your datasets using the following command line:
```
## single file
smbclient -U username%passward //helabdata2.uchicago.edu/your_directory -c 'put local_file remote_file'
## directory
smbclient -U username%passward //helabdata2.uchicago.edu/your_directory -c 'recurse; prompt; mput local_directory*'
## for example
smbclient -U username%passward //helabdata1.uchicago.edu/xcui -c 'put m6A_IP.fq.gz m6A_IP.fq.gz'
```
Please find *Scott* if you don't have an account in our databackup server at <helabdata2.uchicago.edu>.
If you have question about using **smbclient** to upload and download data, please consult *Xiaolong*. 

## Map the data to the mycoplasma genome to check for potential comtamination.
We are going to use fast and sensitive alignment program **Hisat2** to align the reads to the mycoplasma genome. You can go to official page of [Hisat2](https://ccb.jhu.edu/software/hisat2/index.shtml) to learn more about the software and parameters setting.  
  
Mapping can be done at command line by excuting the Hisat2 command one by one for each sample. Alternatively, we use a bash file to wrap the Hisat2 command to loop through all our samples to save us some effort to keep excuting hisat2 command.  
  
You can use any text editor to make the bash file and save it to **something.sh**. For example, in unix command line, use vim to creat and edit `vi run_alignment.sh` and type `i` to insert content. After entering the content, hit `Esc` to quit insertion mode and `Shift` + `:` + `wq` + `Enter` to save and quit.  
  
An example bash file is shown below. You use this template and repalce the sample names by yours. 
```
INDEX="$HOME/Database/genome/contamination/mycoplasma"
Data="$HOME/path_to/your_favorite_directory"

mySample="sample1 sample2 sample3 sample4 treated1 treated2 treated3 treated4 treated5"
for s in $mySample
do 

hisat2 -x $INDEX -k 7 --un-gz $s.IN.noMyco.fastq.gz --summary-file $s.IN.myco_summary -p 4 -U $Data/$s.IN.fastq.gz > $s.myco.input.sam

hisat2 -x $INDEX -k 7  --un-gz $s.m6A.noMyco.fastq.gz --summary-file $s.m6A.myco_summary -p 4 -U $Data/$s.m6A.fastq.gz > $s.myco.m6A.sam
## remove alignment result files because we don't really need alignment to mycoplasma, we only want to see alignment summary
rm $s.myco.input.sam
rm $s.myco.m6A.sam 

wait
done
```
Then excute the bash file in the command line by  
`bash run_alignment.sh`  
  
The code above will align reads to mycoplasma genome and output alignment summary. You can check the alignment summary to assess the potential contamination by Mycoplasma.  
It will also output reads that don't align to mycoplasma nameed as *samplename.IN.noMyco.fastq.gz*. This will be unmmaped reads filtering out mycoplasma reads, which will be input for next round of alignment to human/mouse/whatever_organism reference genome.

## Map the data to the reference genome.
Similar to the code above, we use the **Hisat2** to align the reads to the reference genome of the organism you are studying. The commonly used reference genome for Human and Mouse are downloaded and available at `$HOME/Database/genome` directory. I have also prepared the hisat2 index for **hg38/19** **mm10** so that you can directly use. If you need to use reference genome of other organism, please refer to the [hisat2 mannual](https://ccb.jhu.edu/software/hisat2/manual.shtml) to download and build the index.  
An example of bash file to map to the human hg38 reference genome is provided below:  
```
INDEX="$HOME/Database/genome/hg38/hg38_UCSC"
SPLICE="$HOME/Database/genome/hg38/hisat2_splice_sites.txt"
Data="$HOME/path_to/the_directory_where_sample.IN/m6A.noMyco.fastq.gz_locate"
Output="/path_to_output_directory"

mySample="sample1 sample2 sample3 sample4 treated1 treated2 treated3 treated4 treated5"
for s in $mySample
do 

hisat2 -x $INDEX --known-splicesite-infile $SPLICE -k 1 --no-unal --summary-file $s.IN.align_summary -p 4 -U $Data/$s.IN.noMyco.fastq.gz |samtools view -bS |samtools sort -o $Output/$s.input.bam

hisat2 -x $INDEX --known-splicesite-infile $SPLICE -k 1  --no-unal --summary-file $s.m6A.align_summary -p 4 -U $Data/$s.m6A.noMyco.fastq.gz | samtools view -bS |samtools sort -o $Output/$s.m6A.bam

wait
done
mkdir alignment_summary
mv *.align_summary alignment_summary/
```
Excuting the bash file above will align the reads to hg38 reference genome and out put bam (binary format of sam) files named as **sample.input.bam** and **sample.m6A.bam** in the output directory defined in the bash file.  

With the aligned bam files, you are ready to proceed with varies down stream analysis such as **Differential gene expression** , **Peak calling** or **Differential methylation** analysis.  

## Gene level differential expression analysis can refer to tutorial [here](https://www.bioconductor.org/help/workflows/rnaseqGene/). 

## Peak calling
### 1. Fisher's excat test to call peak
The following tutorial will assume that you store the mapped bam files in directory `/home/xxx/project1/bam_files` and the bam files are named by convention `samplename.input.bam and samplename.m6A.bam`.  
First in a terminal (connected to our analysis server 128.135.225.178), enter R session by type `R` and `Enter`. You should see something like this when you entered R session. 
```

R version 3.4.3 (2017-11-30) -- "Kite-Eating Tree"
Copyright (C) 2017 The R Foundation for Statistical Computing
Platform: x86_64-pc-linux-gnu (64-bit)

R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.

  Natural language support but running in an English locale

R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.

Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.
```

Now you can load the R package **MeRIPtools** for peak calling.  
```
library(MeRIPtools)
```
Then define the sample names of your analysis and the count the reads in consecutive bins on transcript. 
```
samplenames = c("sample1","sample2","sample3")

TestRun <- countReads(samplenames = samplenames,
                      gtf = "~/Database/genome/hg38/hg38_UCSC.gtf",
                      bamFolder = "/home/xxx/project1/bam_files",
                      outputDir =  "/home/xxx/project1/fisherPeak",
                      modification = "m6A",
                      binSize = 50,
                      threads = 6
)

```

Then countReads function will count reads on continuous bins of size defined by `binSize = 50`. The parameter `gtf = "~/Database/genome/hg38/hg38_UCSC.gtf"` defines the annotation file for gene model. If you are working on mouse, thie should be `gtf = "~/Database/genome/mm10/mm10_UCSC.gtf"` (you might need to download gtf files yourself for other organisms).  
The parameter `bamFolder = "/path"` defines the directory where you have stored your bam files. This is related to the previous step `Output="/path_to_output_directory"` where you defined the directory to store aligned BAM files.   

**Note** `MeRIPtools` requires paired **BAM** files for both **INPUT** and **IP**. For each `samplenames` given to the `countReads` function, you will need to have *samplename.input.bam* and *samplename.m6A.bam* in your *bamFolder* directory. 

Next we call peak by
```
TestRun <- callPeakFisher(TestRun, min_counts = 15, peak_cutoff_fdr = 0.05 , peak_cutoff_oddRatio = 1, threads = 6)
```
You can play with different parameter to tune the stringency of peak calling. The `callPeakFisher()` function call peak on each sample separately and add a matrix of logical value to list `TestRun` indicating for each samples (column), whether a bin (row) is enriched or not. You can check the logical values by `TestRun@peakCallResult`.  

If you are interested in getting read counts in each peak (e.g. to estimate enrichment of IP experiment), you can use 
```
TestRun <- JointPeakCount(TestRun, joint_threshold = 2)
```
The parameter `joint_threshold = 2` defines the least number of sample to have peak called at each locus to call this locus as joint peak. If you require a peak to consistently be called in all samples, you should use `joint_threshold = #of samples`. If you want to include more locus where peak is call only in a subset of samples, this parameter difines the number of sample in the subset. Default is 2. The counts of **Input** is store at `TestRun@jointPeak_input` and the counts of **IP** is at `TestRun@jointPeak_ip`.  

To report joint peak of all samples 
```
Joint_peak <- reportJointPeak(TestRun, threads = 6)
unique.Joint_peak <- Joint_peak[which(!duplicated(paste(Joint_peak$chr,Joint_peak$start,Joint_peak$end,sep = ":"))),]
write.table(unique.Joint_peak, file = "/home/<your account name>/project1/fisherPeak/Joint_peak.bed", sep = "\t", row.names = F, col.names = F, quote = F)
```
Please refer to the R package [MeRIPtools](https://scottzijiezhang.github.io/MeRIPtoolsManual/) for more details about peak calling.


### 2. Peak calling using published method [MeTPeak](https://github.com/compgenomics/MeTPeak),[Reference](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4908365/)
```
library(MeTPeak)

samplenames = c("sample1","sample2","sample3")
Input_bamPath <- paste0("/home/xxx/project1/bam_files/",samplenames,".input.bam")
IP_bamPath <- paste0("/home/xxx/project1/bam_files/",samplenames,".m6A.bam")

metpeak(IP_BAM = IP_bamPath, INPUT_BAM = Input_bamPath,
        GENE_ANNO_GTF= "~/Database/genome/hg38/hg38_UCSC.gtf",
        OUTPUT_DIR= "/home/xxx/project1/fisherPeak",
        EXPERIMENT_NAME = "You name it",
        FRAGMENT_LENGTH = 150
        )

```


### Denovo motif search for peaks
In unix shell, `cd` to directory where your `peak.bed` file locate.  First extract peak sequence from reference genome using bed file of peaks. -
```
bedtools getfasta -fi ~/Database/genome/hg38/hg38_UCSC.fa -bed Joint_peak.bed -fo Joint_peak.fa -split -s
```
Please read [**bedtools**](http://bedtools.readthedocs.io/en/latest/content/tools/getfasta.html) documentation to understand what this line of code is doing.

Then we can use motif search tools to perform motif enrichment analysis. Here I will demonstrate using [**Homer**](http://homer.ucsd.edu/homer/index.html) to search for motif. 
```
findMotifs.pl jointPeak.fa fasta /home/xxx/project1/fisherPeak/homer_motif -fasta ~/Database/transcriptome/backgroud_peaks/hg38_200bp_randomPeak.fa -rna -p 10 -len 5,6,7 
```
`jointPeak.fa` is the peak sequence you extracted In previous step in fasta format. `fasta` defined the format of input file. `/home/xxx/project1/fisherPeak/homer_motif` defined the output directory of homer motif analysis. `-fasta` difined the background sequence in fasta format. Here we provide a randomly sample 200bp peaks as background. `-rna` tell homer to output "U" instead of "T". `-p 10` tell homer to use 10 threads. `-len 5,6,7` tell homer to search for motif of length 5,6,7.  

### Plot Meta Gene of the peak you called. 
Here I use R package [Guitar]() to plot the meta gene distribution of peaks. Guitar normalized the density of peaks on each feature (5UTR, CDS, 3UTR) by the length of the feature, which correlates to the expected occurence of peak by chance.  
To get start, we need to have the peak in bed12 format read into R. In previous peak calling step, if you used MeRIPtools to call joint peak:
```
Joint_peak <- reportJointPeak(TestRun, threads = 6)
```
You already have your peak in variable `Joint_peak`. You can proceed to next step
```
plotMetaGene(Joint_peak,gtf = "~/Database/genome/hg38/hg38_UCSC.gtf")
```

If you used other tools to call peak, you can first read peak file into R and then plot the meta gene:
```
peak <- read.table("path/to/peak.bed",sep = "\t", stringsAsFactors = F)
plotMetaGene(peak,gtf = "~/Database/genome/hg38/hg38_UCSC.gtf")
```


## Differential methylation analysis

[Click here](https://scottzijiezhang.github.io/RADARmannual)
