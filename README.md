# Assignment-3-for-Bioinformatics
install.packages("tidyverse")
.cran_packages <- c("cowplot", "picante", "vegan", "HMP", "dendextend", "rms", "devtools")
.bioc_packages <- c("phyloseq", "DESeq2", "microbiome", "metagenomeSeq", 
                    "ALDEx2")
.inst <- .cran_packages %in% installed.packages()
if(any(!.inst)) {install.packages(.cran_packages[!.inst])}
if (!requireNamespace("BiocManager", quietly = TRUE))
install.packages("BiocManager")  
BiocManager::install(.bioc_packages)
devtools::install_github("adw96/breakaway")
#the code above allows us to install all the packages we will need for this
#assignment
#the next lines of code will allow us to load all the packages
library(tidyverse); packageVersion("tidyverse")
library(phyloseq); packageVersion("phyloseq")
library(DESeq2); packageVersion("DESeq2")
library(microbiome); packageVersion("microbiome")
library(vegan); packageVersion("vegan")
library(picante); packageVersion("picante")
library(ALDEx2); packageVersion("ALDEx2")
library(metagenomeSeq); packageVersion("metagenomeSeq")
library(HMP); packageVersion("HMP")
library(dendextend); packageVersion("dendextend")
library(rms); packageVersion("rms")
library(breakaway); packageVersion("breakaway")
#this bit of code allowed us to put the data we need to analyze into our environment
(ps <- readRDS("ps_giloteaux_2016.rds"))
#now we will sort samples on total read counts, remove <5k reads, and remove any OTUs seen in only those samples
sort(phyloseq::sample_sums(ps))
(ps <- phyloseq::subset_samples(ps, phyloseq::sample_sums(ps) > 5000))
(ps <- phyloseq::prune_taxa(phyloseq::taxa_sums(ps) > 0, ps))
#Now we will assign a new sample metadata field
phyloseq::sample_data(ps)$Status <- ifelse(phyloseq::sample_data(ps)$Subject == "Patient", "Chronic Fatigue", "Control")
phyloseq::sample_data(ps)$Status <- factor(phyloseq::sample_data(ps)$Status, levels = c("Control", "Chronic Fatigue"))
ps %>% 
  sample_data %>%
  dplyr::count(Status)
#based on the code above we can see that we have filled our environment with
#phyloseq data that contains 138 taxa on 84 samples, 22 sample metadata fields,
#7 taxonomic ranks and that a phylogenetic tree and the reference sequences have 
#been included. We can also see that there are data on n=37 controls and n=47 
#patients suffering from chronic fatigue
#The next bit of code will allow us to Visualize Relative Abundance
#we want to firs visualize the abundance of organisms at specific taxonomic
#ranks. We can do this by using stacked bar plots and faceted box plots
table(phyloseq::tax_table(ps)[, "Phylum"])
#now we will convert the information to relative abundance
ps_rel_abund = phyloseq::transform_sample_counts(ps, function(x){x / sum(x)})
phyloseq::otu_table(ps)[1:5 , 1:5]
phyloseq::otu_table(ps_rel_abund)[1:5 , 1:5]
#Plot
phyloseq::plot_bar(ps_rel_abund, fill = "Phylum") + 
  geom_bar(aes(color = Phylum, fill = Phylum), stat = "identity", position = "stack") + labs(x = "", y = "Relative bundance\n") + facet_wrap(~ Status, 
                                                                                                                                             scales = "free") + 
  theme(panel.background = element_blank(), axis.text.x = element_blank(), axis.ticks.x = element_blank())
#Based on the results of this code and the plot we acquired, we can see that
#there are nine phyla and their relative abundance looks to be quite similar between groups. No we will generate box plots according to group and facet 
#them by phylum using the raw counts
ps_phylum <- phyloseq::tax_glom(ps, "Phylum")
phyloseq::taxa_names(ps_phylum) <- phyloseq::tax_table(ps_phylum)[, "Phylum"]                                
phyloseq::otu_table(ps_phylum)[1.5 , 1.5]
phyloseq::psmelt(ps_phylum) %>% #shortcut to make pipes shortcut Command + 
  #Shift + M
ggplot(data = ., aes(x = Status, y = Abundance)) +  geom_boxplot(outlier.shape = NA) + geom_jitter(aes(color = OTU), height = 0, width = 0.2) + labs(x = "", y = "Abundance\n") + facet_wrap(~ OTU, scales = "free")
#based on the results we can see that any samples have a higher number of Firmicutes, followed by Bacteriodetes and Actinobacteria. Most samples have low read counts for other phyla with some outlying samples. There doesnt appear to be much difference in the major phyla between groups
#Subset groups
controls <- phyloseq::subset_samples(ps_phylum, Status == "Control")
cf <- phyloseq::subset_samples(ps_phylum, Status == "Chronic Fatigue")
#Output OTU tables
control_otu <- data.frame(phyloseq::otu_table(controls))
cf_otu <- data.frame(phyloseq::otu_table(cf))
#Group rare phyla
control_otu <- control_otu %>%
t(.) %>% #transpose 
as.data.frame(.) %>%  
mutate(Other = Cyanobacteria + Euryarchaeota + Tenericutes + Verrucomicrobia + Fusobacteria) %>%  
dplyr::select(-Cyanobacteria, -Euryarchaeota, -Tenericutes, -Verrucomicrobia, -Fusobacteria)  
cf_otu <- cf_otu %>%
  t(.) %>%
  as.data.frame(.) %>%
  mutate(Other = Cyanobacteria + Euryarchaeota + Tenericutes + Verrucomicrobia + Fusobacteria) %>%
  dplyr::select(-Cyanobacteria, -Euryarchaeota, -Tenericutes, -Verrucomicrobia, -Fusobacteria)  
#HMP test
group_data <- list(control_otu, cf_otu)
(xdc <- HMP::Xdc.sevsample(group_data))
1 - pchisq(0.2769004, 5)
#Hierarchical Clustering based on Bray-Curtis Dissimilarity
#as two samples are fewer taxa, the number increases
#first we will extract eh OTU table and compute BC
ps_rel_otu <- data.frame(phyloseq::otu_table(ps_rel_abund))
ps_rel_otu <- t(ps_rel_otu)
bc_dist <- vegan::vegdist(ps_rel_otu, method = "bray")
as.matrix(bc_dist) [1:5, 1:5]
#save as dendrogram
ward <- as.dendrogram(hclust(bc_dist, method = "ward.D2"))
#Provide color codes
meta <- data.frame(phyloseq::sample_data(ps_rel_abund))
colorCode <- c(Control = "green", 'Chronic Fatigue' = "blue")
labels_colors(ward) <- colorCode[meta$Status] [order.dendrogram(ward)]
#Plot
plot(ward)
#Alpha Diversity: it is a within sample diversity that includes the number of organisms observed and how evenly they are distributed
ggplot(data = data.frame("total_reads" = phyloseq::sample_sums(ps), "observed" = phyloseq::estimate_richness(ps, measures = "Observed") [, 1]), aes(x = total_reads, y = observed)) + 
         geom_point() + 
         geom_smooth(method = "lm", se = FALSE) + labs(x = "\nTotal Reads", y = "Observed Richness\n")
## 'geom_smooth()' using formula 'y ~ x'      
#Subsample reads
(ps_rare <- phyloseq::rarefy_even_depth(ps, rngseed = 123, replace = FALSE))
head(phyloseq::sample_sums(ps_rare))
#generate a data.frame with adiv frames
adiv <- data.frame("Observed" = phyloseq::estimate_richness(ps_rare, measures = "Observed"), 
                   "Shannon" = phyloseq::estimate_richness(ps_rare, measures = "Shannon"), 
                   "PD" = picante::pd(samp = data.frame(t(data.frame(phyloseq::otu_table(ps_rare)))), tree = phyloseq::phy_tree(ps_rare))[, 1], 
                   "Status" = phyloseq::sample_data(ps_rare)$Status)
head(adiv)
#Plot adiv measures
adiv %>%
  gather(key = metric, value = value, c("Observed", "Shannon", "PD")) %>%
  mutate(metric = factor(metric, levels = c("Observed", "Shannon", "PD"))) %>%
  ggplot(aes(x = Status, y = value)) +
  geom_boxplot(outlier.color = NA) +
  geom_jitter(aes(color = Status), height = 0, width = .2) +
  labs(x = "", y = "") +
  facet_wrap(~ metric, scales = "free") +
  theme(legend.position="none")
#Summarize
adiv %>%
  group_by(Status) %>%
  dplyr::summarise(median_observed = median(Observed),
                   median_shannon = median(Shannon),
                   median_pd = median(PD))
#Wilcoxon test of location
wilcox.test(Observed ~ Status, data = adiv, exact = FALSE, conf.int = TRUE)
wilcox.test(Shannon ~ Status, data = adiv, conf.int = TRUE)      
wilcox.test(PD ~ Status, data = adiv, conf.int = TRUE)
#Obtain breakaway estimates
ba_adiv <- breakaway::breakaway(ps)
ba_adiv[1]
#Plot estimates
      plot(ba_adiv, ps, color = "Status")     
#Examine models
summary(ba_adiv) %>%
  add_column("SampleNames" = ps %>% otu_table %>% sample_names)  
#Test for group difference
bt <- breakaway::betta(summary(ba_adiv)$estimate, summary(ba_adiv)$error, make_design_matrix(ps, "Status"))
bt$table
#BETA-DIVERSITY: provides a measure of similarity, or dissimilarity, of one microbial composition to another
#CLR transform
(ps_clr <- microbiome::transform(ps, "clr"))
phyloseq::otu_table(ps)[1:5, 1:5]
phyloseq::otu_table(ps_clr)[1:5, 1:5]
#PCA via phyloseq
ord_clr <- phyloseq::ordinate(ps_clr, "RDA")
#Plot scree plot
phyloseq::plot_scree(ord_clr) + 
  geom_bar(stat = "identity", fill = "green") + 
  labs(x = "\nAxis", y = "proportion of Variance\n")
#Examine eigenvalues and % prop. Variance explained
head(ord_clr$CA$eig)
sapply(ord_clr$CA$eig[1:5], function(x), x / sum(ord_clr$CA$eig))
#Scale axes and plot ordination
clr1 <- ord_clr$CA$eig[1] / sum(ord_clr$CA$eig)
clr2 <- ord_clr$CA$eig[2] / sum(ord_clr$CA$eig)
phyloseq::plot_ordination(ps, ord_clr, type="samples", color="Status") + 
  geom_point(size = 2) +
  coord_fixed(clr2 / clr1) +
  stat_ellipse(aes(group = Status), linetype = 2)
#Generate distance matrix
clr_dist_matrix <- phyloseq::distance(ps_clr, method = "euclidean")
#ADONIS test
vegan::adonis(clr_dist_matrix ~ phyloseq::sample_data(ps_clr)$Status)
#Dispersion test and plot
dispr <- vegan::betadisper(clr_dist_matrix, phyloseq::sample_data(ps_clr)$Status)
dispr                           
plot(dispr, main = "Ordination Centroids and Dispersion Labeled: Aitchison Distance", sub = "")
boxplot(dispr, main = "", xlab = "")
permutest(dispr)
#Generate Distances
ord_unifrac <- ordinate(ps_rare, method = "PCoA", distance = "wunifrac") 
ord_unifrac_un <- ordinate(ps_rare, method = "PCoA", distance = "unifrac")  
#Plot ordinations
a <- plot_ordination(ps_rare, ord_unifrac, color = "Status") + geom_point(size = 2)
b <- plot_ordination(ps_rare, ord_unifrac_un, color = "Status") + geom_point(size = 2)
cowplot::plot_grid(a, b, nrow = 1, ncol = 2, scale = .9, labels = c("Weighted", "Unweighted"))
#Differential Abundance Testing: to identify specific taxa associated with clinical metadata variables of interest
#Generate data.frame with OTUs and metadata
ps_wilcox <- data.frame(t(data.frame(phyloseq::otu_table(ps_clr))))
ps_wilcox$Status <- phyloseq::sample_data(ps_clr)$Status
#Define functions to pass to map
wilcox_model <- function(df){
  wilcox.test(abund ~ Status, data = df)
}
wilcox_pval <- function(df){
  wilcox.test(abund ~ Status, data = df)$p.value
}
#Create nested data frames by OTU and loop over each using map 
wilcox_results <- ps_wilcox %>%
  gather(key = OTU, value = abund, -Status) %>%
  group_by(OTU) %>%
  nest() %>%
  mutate(wilcox_test = map(data, wilcox_model),
         p_value = map(data, wilcox_pval))                       
#Show results
head(wilcox_results)
head(wilcox_results$data[[1]])
wilcox_results$wilcox_test[[1]]
#Unnesting
wilcox_results <- wilcox_results %>%
  dplyr::select(OTU, p_value) %>%
  unnest()
head(wilcox_results)
#Adding taxonomic labels
taxa_info <- data.frame(tax_table(ps_clr))
taxa_info <- taxa_info %>% rownames_to_column(var = "OTU")
#Computing FDR corrected p-values
wilcox_results <- wilcox_results %>%
  full_join(taxa_info) %>%
  arrange(p_value) %>%
  mutate(BH_FDR = p.adjust(p_value, "BH")) %>%
  filter(BH_FDR < 0.05) %>%
  dplyr::select(OTU, p_value, BH_FDR, everything())
#Printing results
print.data.frame(wilcox_results)
#Run ALDEx2
aldex2_da <- ALDEx2::aldex(data.frame(phyloseq::otu_table(ps)), phyloseq::sample_data(ps)$Status, test="t", effect = TRUE, denom="iqlr")
#Plot effect sizes
ALDEx2::aldex.plot(aldex2_da, type="MW", test="wilcox", called.cex = 1, cutoff = 0.05)
#Clean up presentation
sig_aldex2 <- aldex2_da %>%
  rownames_to_column(var = "OTU") %>%
  filter(wi.eBH < 0.05) %>%
  arrange(effect, wi.eBH) %>%
  dplyr::select(OTU, diff.btw, diff.win, effect, wi.ep, wi.eBH)
sig_aldex2 <- left_join(sig_aldex2, taxa_info)
sig_aldex2
#PREDICTION
#Generate data.frame
clr_pcs <- data.frame(
  "pc1" = ord_clr$CA$u[,1],
  "pc2" = ord_clr$CA$u[,2],
  "pc3" = ord_clr$CA$u[,3],
  "Status" = phyloseq::sample_data(ps_clr)$Status
)
clr_pcs$Status_num <- ifelse(clr_pcs$Status == "Control", 0, 1)
head(clr_pcs)
#Specify a datadist object (for rms)
dd <- datadist(clr_pcs)
options(datadist = "dd")
#Plot the unconditional associations
a <- ggplot(clr_pcs, aes(x = pc1, y = Status_num)) +
  Hmisc::histSpikeg(Status_num ~ pc1, lowess = TRUE, data = clr_pcs) +
  labs(x = "\nPC1", y = "Pr(Chronic Fatigue)\n")
b <- ggplot(clr_pcs, aes(x = pc2, y = Status_num)) +
  Hmisc::histSpikeg(Status_num ~ pc2, lowess = TRUE, data = clr_pcs) +
  labs(x = "\nPC2", y = "Pr(Chronic Fatigue)\n")
c <- ggplot(clr_pcs, aes(x = pc3, y = Status_num)) +
  Hmisc::histSpikeg(Status_num ~ pc3, lowess = TRUE, data = clr_pcs) +
  labs(x = "\nPC3", y = "Pr(Chronic Fatigue)\n")
cowplot::plot_grid(a, b, c, nrow = 2, ncol = 2, scale = .9, labels = "AUTO")
#Fit full model with splines (3 knots each)
m1 <- rms::lrm(Status_num ~ rcs(pc1, 3) + rcs(pc2, 3) + rcs(pc3, 3), data = clr_pcs, x = TRUE, y = TRUE)
#Grid search for penalties
pentrace(m1, list(simple = c(0, 1, 2), nonlinear = c(0, 100, 200)))
pen_m1 <- update(m1, penalty = list(simple = 1, nonlinear = 200))
pen_m1
#Plot log odds
ggplot(Predict(pen_m1))
#Obtain optimism corrected estimates
(val <- rms::validate(pen_m1))
#Compute corrected c-statistic
(c_opt_corr <- 0.5 * (val[1, 5] + 1))
#Plot calibration
cal <- rms::calibrate(pen_m1, B = 200)
plot(cal)
#Output pred. probs
head(predict(pen_m1, type ="fitted"))

#RANDOM FOREST
install.packages("randomForest")
table <- t(ps_rare@otu_table)
forest <- randomForest(x = table, y = ps_rare@sam_data$Status, ntree = 1000, importance = TRUE)
forest
print(forest)
importance(forest)
#OTU 1 is most important
#OOB error rate is 26.19%
#Family: Porphyromonadaceae, Genus: Parabacteroides


#Combining estimation and summarizing steps of Alpha-Diversity
adiv <- function(df){ 
  estimates <- data.frame(
    "Observed" = phyloseq::estimate_richness(df, measures = "Observed"),
    "Shannon" = phyloseq::estimate_richness(df, measures = "Shannon"),
    "PD" = picante::pd(samp = data.frame(t(data.frame(phyloseq::otu_table(df)))), tree = phyloseq::phy_tree(df))[, 1],
    "Status" = phyloseq::sample_data(df)$Status)
  summary <- estimates %>%
    group_by(Status) %>%
    dplyr::summarise(median_observed = median(Observed),
                              median_shannon = median(Shannon),
                              median_pd = median(PD))
  print(summary)
  return(estimates)}

poop <- adiv(ps_rare)

#In order to write this code we assigned a function within a function, then copied the code from earlier. Used the return function with the assigned name 
#and then printed a summary.
