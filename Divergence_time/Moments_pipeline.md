# Moments analysis
## Process the data
### Remove known inversions and duplicates from the bam files
```
bedtools subtract -a ~/autosomes.bed -b mask.bed > keep_for_sfs.bed
samtools view -b -h -L ~/keep_for_sfs.bed ~/${pop}/${sample}_realgn_sorted.bam > ~/sfs_bams/${pop}/${sample}_sfs.bam
```
### Filter data with ANGSD
```
ls ~/sfs_bams/${pop}/*.bam > ~/sfs/${pop}.bamlist

REF=~/reference/NSP_V6b.fasta
FILTERS="-uniqueOnly 1 -remove_bads 1  -skipTriallelic 1 -minMapQ 20 -minQ 20 -doHWE 1 -C 50"
TODO="-doMajorMinor 1 -doMaf 1 -dosnpstat 1 -doPost 2 -doGeno 11"

nind=$(awk '{x++} END {print x}' "${pop}.bamlist")
angsd -b ${pop}.bamlist -ref $REF -GL 1 -P 10 $FILTERS $TODO -minInd ${nind}  -out ~/sfs/${pop}/${pop}_sfilt &>> ~/sfs/${pop}/${pop}_sfilt.log
```
### Filter out sites where heterozygote counts comprise more than 70% of all counts (likely lumped paralogs)
```
zcat ~/sfs/${pop}/${pop}_sfilt.snpStat.gz | awk '($3+$4+$5+$6)>0' | awk '($12+$13+$14+$15)/($3+$4+$5+$6)<0.7' | cut -f 1,2  > ~/sfs/${pop}/${pop}.sites2do
angsd sites index ~/sfs/${pop}/${pop}.sites2do
```
## Get folded 1d SFS by set outgroup with reference genome
``
TODO2="-doSaf 1 -anc $REF -ref $REF"
angsd -b ${pop}.bamlist -GL 1 -P 10 -sites ~/sfs/${pop}/${pop}.sites2do $TODO2 -out ~/sfs/${pop}/${pop} &>>  ~/sfs/${pop}/${pop}_saf.log
``

## Get 2dSFS
### Unify the sites used for calculation
```
awk '{print $1"_"$2}' ~/sfs/${pop1}/${pop1}.sites2do > ~/sfs/joint_sfs_2d/${pop1}.sites 
awk '{print $1"_"$2}' ~/sfs/${pop2}/${pop2}.sites2do > ~/sfs/joint_sfs_2d/${pop2}.sites 
comm -12 <(sort -T ~/tmp -n ~/sfs/joint_sfs_2d/${pop1}.sites) <(sort -T ~/tmp2 -n ~/sfs/joint_sfs_2d/${pop2}.sites) > ~/sfs/joint_sfs_2d/${pop1}_${pop2}.sites 
awk -F "_" 'BEGIN {OFS="\t"}; {print $1,$2}' ~/sfs/joint_sfs_2d/${pop1}_${pop2}.sites |sort -V > ~/sfs/joint_sfs_2d/${pop1}_${pop2}.sites2do 
angsd sites index ~/sfs/joint_sfs_2d/${pop1}_${pop2}.sites2do 
```
### Estimate the 2dSFS
```
nindpop1=$(awk '{x++} END {print x}' "~/sfs/${pop1}.bamlist")
nindpop2=$(awk '{x++} END {print x}' "~/sfs/${pop2}.bamlist")
REF=~/reference/NSP_V6b.fasta
realSFS ~/sfs/${pop1}/${pop1}.saf.idx -sites ~/sfs/joint_sfs_2d/${pop1}_${pop2}.sites2do -maxIter 100 -P 16 > ~/sfs/joint_sfs_2d/SFS/${pop1}_vs_${pop2}_2d_${pop1}.sfs 2> ~/sfs/joint_sfs_2d/SFS/${pop1}_vs_${pop2}_2d_${pop1}.sfs.log
realSFS ~/sfs/${pop2}/${pop2}.saf.idx -sites ~/sfs/joint_sfs_2d/${pop1}_${pop2}.sites2do -maxIter 100 -P 16 > ~/joint_sfs_2d/SFS/${pop1}_vs_${pop2}_2d_${pop2}.sfs 2> ~/sfs/joint_sfs_2d/SFS/${pop1}_vs_${pop2}_2d_${pop2}.sfs.log
realSFS dadi -P 16 -sites ~/sfs/joint_sfs_2d/${pop1}_${pop2}.sites2do ~/sfs/${pop1}/${pop1}.saf.idx ~/sfs/${pop2}/${pop2}.saf.idx -sfs ~/sfs/joint_sfs_2d/SFS/${pop1}_vs_${pop2}_2d_${pop1}.sfs -sfs ~/sfs/joint_sfs_2d/SFS/${pop1}_vs_${pop2}_2d_${pop2}.sfs -ref $REF -anc $REF > ~/sfs/joint_sfs_2d/2dsfs/${pop1}_${pop2}_2d.dadi.sfs 2> ~/sfs/joint_sfs_2d/2dsfs/${pop1}_${pop2}_2d.dadi.sfs.log
realSFS -P 16 -sites ~/sfs/joint_sfs_2d/${pop1}_${pop2}.sites2do ~/sfs/${pop1}/${pop1}.saf.idx  ~/sfs/${pop2}/${pop2}.saf.idx -ref $REF -anc $REF > ~/sfs/joint_sfs_2d/2dsfs/${pop1}_${pop2}_2d.sfs 2> ~/sfs/joint_sfs_2d/2dsfs/${pop1}_${pop2}_2d.sfs.log
```
### Convert to dadi format, the perl script can be found [here](https://github.com/z0on/2bRAD_denovo/blob/master/realsfs2dadi.pl)
```
perl ~/sfs/realsfs2dadi.pl ~/sfs/joint_sfs_2d/2dsfs/${pop1}_${pop2}_2d.dadi.sfs ${nindpop1} ${nindpop2} > ~/sfs/joint_sfs_2d/2dsfs/${pop1}_${pop2}.dadi.data
```
## Run moments with Slurm, python scripts by [Paolo Momigliano](https://github.com/Nopaoli) can be found [here](https://github.com/XueyunF/Phylogeographic-analysis-of-nine-spined-sticklebacks/tree/main/Estimation%20of%20the%20divergence%20times%20and%20data%20simulation)
```
MODEL=$(sed -n ${SLURM_ARRAY_TASK_ID}p ~/moments/2dsfs/Runs2 | cut -f 1)
REP=$(sed -n ${SLURM_ARRAY_TASK_ID}p ~/moments/2dsfs/Runs2 | cut -f 2)
python ~/moments/Script_moments_model_optimisation.py -f pop1_pop2.dadi -y Pop1 -x Pop2 -p 40,40 -s True -n 6 -r $REP -m $MODEL
```
### Summarise the results
```
mv ~/moments/2dsfs/.optimized.txt ~/moments/2dsfs/optimized
python ~/moments/Summarize_Outputs.py ~/moments/2dsfs/optimized
```
### Reshape the results format
```
sed 's/,/\t/g' ~/moments/2dsfs/optimized/Results_Summary_Short_${poppair}.txt > Results_Summary_Short_edited_${poppair}.txt
cat Results_Summary_Short_edited_${poppair}.txt|grep -w "IM" |sed -e '1i\Model Replicate log-likelihood AIC chi-squared theta nu1 nu2 T1 m12 m21' > ${poppair}_IM
cat Results_Summary_Short_edited_${poppair}.txt|grep -w "SI" |sed -e '1i\Model Replicate log-likelihood AIC chi-squared theta nu1 nu2 T1' > ${poppair}_SI
```
## Scale the parameters in R
```
#"Length" is a file has the population pair name and total number of SNPs in the pop1_pop2.dadi.data file
library(ggplot2)
library(dplyr)
library(cowplot)
library(ggpubr)
pairs<-read.table("pairs_info",header = T)

readpairs=function(pairname){
  result1<-read.table(paste0("$PATH",pairname,"_IM"),header = T)
  result2<-read.table(paste0("$PATH",pairname,"_SI"),header = T)
  odfa<-dplyr::bind_rows(result1,result2)  
  return(odfa)
}
pair1<-readpairs(pairs[1,1])

#Lratio=(sub SNP/ all SNP) * total sites
#mu * L = MU
#since we use all the snps therefore Lratio equals to 1
pairs$Lratio<-1
mu_pair1<-pairs$Lratio[1]*mu*sum(scan("~/sfs/joint_sfs_2d/2dsfs/${pop1}_${pop2}_2d.sfs"))

#if multiple pairs involves
mu2<-mean(c(mupair1,mupair2...mupairN))

#merge the data frame
df<-rbind(pair1,pair2...pairN)
df[is.na(df)] <- 0
df$Nref<-df$theta/(4*mu) ### Ne

df$N1a<- df$nu1*df$Nref ### Get pop size of N2
df$N2a<- df$nu2*df$Nref ### Get pop size of N2
df$N1b<- df$nu1b*df$Nref ### Get pop size of N2
df$N2b<- df$nu2b*df$Nref ### Get pop size of N2
df$M12<-df$m12/(2*df$Nref) ### Migration rates are in units of 2*Nref*mij
df$M21<-df$m21/(2*df$Nref) 
df$M12b<-df$m12_0/(2*df$Nref) 
df$M21b<-df$m21_0/(2*df$Nref) 
df$TtotGen<-df$Ttot*2*df$Nref
#df$TaeGen<-df$TaeTot*2*df$Nref
#df$TaeYear<-df$TaeGen*2
df$T2Gen<-df$T2*2*df$Nref
df$T1Gen<-df$T1*2*df$Nref

generation=2
df$T2Year<-df$T2Gen*generation
df$T1Year<-df$T1Gen*generation
df$TtotYear<-df$TtotGen*generation
df$T12_ratio<-df$T2/df$T1

df$Ttot<-df$T1+df$T2 ### divergence time between pop1 and pop2

df$Nem12b<- df$M21b*df$N2b  ## This is the total number of migrants from pop1 to pop2
df$Nem21b<- df$M12b*df$N1b  ##  This is the number of migrants from pop2 to pop1

df$Nem12a<- df$M21*df$N2a  ## This is the total number of migrants from pop1 to pop2
df$Nem21a<- df$M12*df$N1a  ##  This is the number of migrants from pop2 to pop1

df$delta.aic= df$AIC-min(df$AIC)
```
#### Select the top 5 models based on log.likelihoods
```
best5 <- df %>%
  group_by(Model,POP) %>%
  top_n(5, log.likelihood)
```

#### Visualisation 
```
ggplot(best,aes(x=log.likelihood,y=TtotYear/1000,color=Model,shape=Model))+geom_point(size=5,position=position_jitter(h=0.1, w=0.1))+facet_wrap(~POP, ncol=3)+
  scale_y_continuous(breaks = c(0,50,100,150,200), limits = c(0,200))+theme_bw()+labs(y="Years in Thousand")+ggtitle("Moments anaysis of divergence time")
```
