Process PAAD from cBio
================
Matthew Berginski
5/8/2020

# Load Data

cBioportal provides pre-processed data sets from a wide range of
studies, including the TCGA. I loaded the TCGA pancreatic cancer data
set using the instructions on github
(<https://github.com/cBioPortal/datahub>).

``` r
tcga_paad_CNA = read_delim(here('raw_data/datahub/public/paad_tcga_pan_can_atlas_2018/data_CNA.txt'), delim="\t") %>%
  select(-Entrez_Gene_Id) %>%
  gather('case_id','CNA',-Hugo_Symbol)
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_double(),
    ##   Hugo_Symbol = col_character()
    ## )

    ## See spec(...) for full column specifications.

``` r
tcga_paad_mut = read_delim(here('raw_data/datahub/public/paad_tcga_pan_can_atlas_2018/data_mutations_extended.txt'),
                      delim="\t") %>%
  select(-Entrez_Gene_Id) %>%
  filter(Variant_Classification == "Nonsense_Mutation") %>%
  rename(case_id = Tumor_Sample_Barcode) %>%
  mutate(Nonsense_mutation = TRUE)
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_character(),
    ##   Entrez_Gene_Id = col_double(),
    ##   Start_Position = col_double(),
    ##   End_Position = col_double(),
    ##   t_ref_count = col_double(),
    ##   t_alt_count = col_double(),
    ##   n_ref_count = col_double(),
    ##   n_alt_count = col_double(),
    ##   Protein_position = col_double(),
    ##   Hotspot = col_double(),
    ##   NCALLERS = col_double(),
    ##   n_depth = col_double(),
    ##   t_depth = col_double()
    ## )

    ## See spec(...) for full column specifications.

``` r
tcga_paad_mut_single_cases = tcga_paad_mut %>%
  select(case_id,Nonsense_mutation, Hugo_Symbol) %>%
  group_by(Hugo_Symbol, case_id) %>%
  slice(1) %>%
  ungroup()
```

``` r
tcga_paad_mRNA_z = read_delim(here('raw_data/datahub/public/paad_tcga_pan_can_atlas_2018/data_RNA_Seq_v2_mRNA_median_Zscores.txt'), delim="\t") %>%
  select(-Entrez_Gene_Id) %>%
  gather('case_id','mRNA_z',-Hugo_Symbol) %>%
  filter(!is.na(mRNA_z)) %>%
  group_by(Hugo_Symbol) %>%
  summarise(mRNA_z = mean(mRNA_z,na.rm = T)) %>%
  ungroup()
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_double(),
    ##   Hugo_Symbol = col_character()
    ## )

    ## See spec(...) for full column specifications.

``` r
tcga_paad_data = tcga_paad_CNA %>%
  left_join(tcga_paad_mut_single_cases %>% select(case_id,Nonsense_mutation, Hugo_Symbol)) %>%
  left_join(tcga_paad_mRNA_z) %>%
  mutate(Nonsense_mutation = ifelse(is.na(Nonsense_mutation),FALSE,TRUE)) %>%
  mutate(mRNA_z = ifelse(is.na(mRNA_z),0,mRNA_z)) %>%
  identity()
```

    ## Joining, by = c("Hugo_Symbol", "case_id")

    ## Joining, by = "Hugo_Symbol"

# Determine Essential Gene Pairs

``` r
gene_alteration_rates_paad = tcga_paad_data %>%
  group_by(Hugo_Symbol) %>%
  summarise(del_mut_rate = sum(CNA == -2 | Nonsense_mutation | mRNA_z <= -2)/length(CNA))

ggplot(gene_alteration_rates_paad, aes(x=del_mut_rate)) + 
  geom_histogram(breaks = seq(0,0.35,by=0.01)) +
  xlab('Gene Deletion/Mutation Rate') +
  BerginskiRMisc::theme_berginski()
```

![](process_PAAD_from_cBio_files/figure-gfm/gene%20deletion/alteration%20rates-1.png)<!-- -->

``` r
tcga_paad_altered_matrix = tcga_paad_data %>% 
  mutate(del_mut = ifelse(CNA == -2 | Nonsense_mutation | mRNA_z <= -2, TRUE, FALSE)) %>% 
  select(Hugo_Symbol,case_id,del_mut) %>% 
  pivot_wider(names_from = Hugo_Symbol, values_from = del_mut) %>%
  column_to_rownames(var = "case_id")
```

``` r
source(here('shared_functions.R'))

plan(multisession, workers = availableCores() - 4)

total_samples = length(unique(tcga_paad_data$case_id))

dir.create(here('results/PAAD'), showWarnings = F)
```

## Search for syn lethal pairs

### Min 15% Alteration

``` r
source(here('shared_functions.R'))
tic();
possible_essential_pairs_15 = gene_alteration_rates_paad %>% 
  filter(del_mut_rate >= 0.15) %>%
  
  #produce a list of all the gene permutations
  rename(Hugo_Symbol_1 = Hugo_Symbol) %>%
  mutate(Hugo_Symbol_2 = Hugo_Symbol_1) %>%
  expand(Hugo_Symbol_1,Hugo_Symbol_2) %>% 
  
  #remove the duplicated permutation sets
  filter(Hugo_Symbol_1 < Hugo_Symbol_2) %>%
  rename(gene_1 = Hugo_Symbol_1, gene_2 = Hugo_Symbol_2) %>%

  #this isn't strictly needed, but since PTEN is focus of the analysis, this
  #swaps the location of PTEN to gene_1 when PTEN is in the gene_2 slot and
  #makes looking at the downstream PTEN data easier
  mutate(gene_1_PTEN = ifelse(gene_2 == "PTEN", "PTEN", gene_1),
         gene_2_PTEN = ifelse(gene_2 == "PTEN", gene_1, gene_2)) %>%
  mutate(gene_1 = gene_1_PTEN, gene_2 = gene_2_PTEN) %>%
  select(-gene_1_PTEN, - gene_2_PTEN) %>%

  mutate(overlap_count = future_map2_int(gene_1,gene_2,
                                         find_overlap_count_matrix,
                                         alteration_matrix = tcga_paad_altered_matrix,
                                         .progress=T)) %>%
  
  #merge back in the alteration rates for both gene_1 and gene_2
  left_join(gene_alteration_rates_paad %>% 
              rename(gene_1_mut_rate = del_mut_rate),by=c('gene_1' = 'Hugo_Symbol')) %>%
  left_join(gene_alteration_rates_paad %>% 
              rename(gene_2_mut_rate = del_mut_rate),by=c('gene_2' = 'Hugo_Symbol')) %>%

  #calculate the expected number of overlaps from the base rates
  mutate(expected_overlap_count = gene_1_mut_rate * gene_2_mut_rate * total_samples) %>%
  mutate(overlap_diff = overlap_count - expected_overlap_count) %>%
  mutate(overlap_p_value = future_pmap_dbl(list(overlap_count,total_samples,gene_1_mut_rate*gene_2_mut_rate),
                                           run_binom_test,
                                           .progress = T)) %>%
  mutate(overlap_p_value_adjust = p.adjust(overlap_p_value, method = "fdr")) %>%
  write_rds(here('results/PAAD/all_pairs_15.rds')) %>%
  identity()
```

    ##  Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────── 100%

``` r
toc();
```

    ## 1.633 sec elapsed

``` r
apparent_gene_essential_pairs = possible_essential_pairs_15 %>%
  filter(overlap_diff < 0, overlap_p_value_adjust <= 0.05) %>%
  write_rds(here('results/PAAD/essential_pairs_15.rds'))

apparent_gene_essential_pairs_PTEN = apparent_gene_essential_pairs %>%
  filter(gene_1 == "PTEN" | gene_2 == "PTEN") %>%
  write_rds(here('results/PAAD/PTEN_essential_pairs_15.rds'))
```

``` r
brca_all_pairs_15 = read_rds(here('results/all_pairs_15.rds'))

overlap_diffs = data.frame(cancer = c(rep("PAAD",length(possible_essential_pairs_15$overlap_count)),
                                      rep("BRCA", length(brca_all_pairs_15$overlap_count))),
                           overlap_diffs = c(possible_essential_pairs_15$overlap_diff,
                                             brca_all_pairs_15$overlap_diff)) %>%
  mutate(min_cutoff = 15)

ggplot(overlap_diffs, aes(x=overlap_diffs,color=cancer)) + 
  geom_vline(aes(xintercept = 0), linetype = "dashed") +
  geom_density() + 
  ggtitle(paste0("# Pairs Below 0: PAAD - ", sum(possible_essential_pairs_15$overlap_diff < 0),
                 " BRCA - ", sum(brca_all_pairs_15$overlap_diff < 0))) +
  BerginskiRMisc::theme_berginski()
```

![](process_PAAD_from_cBio_files/figure-gfm/comparison%20to%20BRCA%2015-1.png)<!-- -->

### Min 10% Alteration

``` r
tic();
possible_essential_pairs_10 = gene_alteration_rates_paad %>% 
  filter(del_mut_rate >= 0.10) %>%
  
  #produce a list of all the gene permutations
  rename(Hugo_Symbol_1 = Hugo_Symbol) %>%
  mutate(Hugo_Symbol_2 = Hugo_Symbol_1) %>%
  expand(Hugo_Symbol_1,Hugo_Symbol_2) %>% 
  
  #remove the duplicated permutation sets
  filter(Hugo_Symbol_1 < Hugo_Symbol_2) %>%
  rename(gene_1 = Hugo_Symbol_1, gene_2 = Hugo_Symbol_2) %>%

  #this isn't strictly needed, but since PTEN is focus of the analysis, this
  #swaps the location of PTEN to gene_1 when PTEN is in the gene_2 slot and
  #makes looking at the downstream PTEN data easier
  mutate(gene_1_PTEN = ifelse(gene_2 == "PTEN", "PTEN", gene_1),
         gene_2_PTEN = ifelse(gene_2 == "PTEN", gene_1, gene_2)) %>%
  mutate(gene_1 = gene_1_PTEN, gene_2 = gene_2_PTEN) %>%
  select(-gene_1_PTEN, - gene_2_PTEN) %>%

  #determine the number of cases where alterations overlap
  mutate(overlap_count = future_map2_int(gene_1,gene_2,
                                         find_overlap_count_matrix,
                                         alteration_matrix = tcga_paad_altered_matrix,
                                         .progress=T)) %>%

  #merge back in the alteration rates for both gene_1 and gene_2
  left_join(gene_alteration_rates_paad %>% 
              rename(gene_1_mut_rate = del_mut_rate),by=c('gene_1' = 'Hugo_Symbol')) %>%
  left_join(gene_alteration_rates_paad %>% 
              rename(gene_2_mut_rate = del_mut_rate),by=c('gene_2' = 'Hugo_Symbol')) %>%

  #calculate the expected number of overlaps from the base rates
  mutate(expected_overlap_count = gene_1_mut_rate * gene_2_mut_rate * total_samples) %>%
  mutate(overlap_diff = overlap_count - expected_overlap_count) %>%
  mutate(overlap_p_value = future_pmap_dbl(list(overlap_count,total_samples,gene_1_mut_rate*gene_2_mut_rate),
                                           run_binom_test,
                                           .progress = T)) %>%
  mutate(overlap_p_value_adjust = p.adjust(overlap_p_value, method = "fdr")) %>%
  write_rds(here('results/PAAD/all_pairs_10.rds')) %>%
  mutate_if(is.numeric, funs(signif(.,3))) %>%
  identity()
```

    ## Warning: funs() is soft deprecated as of dplyr 0.8.0
    ## Please use a list of either functions or lambdas: 
    ## 
    ##   # Simple named list: 
    ##   list(mean = mean, median = median)
    ## 
    ##   # Auto named with `tibble::lst()`: 
    ##   tibble::lst(mean, median)
    ## 
    ##   # Using lambdas
    ##   list(~ mean(., trim = .2), ~ median(., na.rm = TRUE))
    ## This warning is displayed once per session.

``` r
toc();
```

    ## 1.017 sec elapsed

``` r
apparent_gene_essential_pairs = possible_essential_pairs_10 %>%
  filter(overlap_diff < 0, overlap_p_value_adjust <= 0.05) %>%
  write_rds(here('results/PAAD/essential_pairs_10.rds'))

apparent_gene_essential_pairs_PTEN = apparent_gene_essential_pairs %>%
  filter(gene_1 == "PTEN" | gene_2 == "PTEN") %>%
  write_rds(here('results/PAAD/PTEN_essential_pairs_10.rds'))
```

``` r
brca_all_pairs_10 = read_rds(here('results/all_pairs_10.rds'))

overlap_diffs = data.frame(cancer = c(rep("PAAD",length(possible_essential_pairs_10$overlap_count)),
                                      rep("BRCA", length(brca_all_pairs_10$overlap_count))),
                           overlap_diffs = c(possible_essential_pairs_10$overlap_diff,
                                             brca_all_pairs_10$overlap_diff)) %>%
  mutate(min_cutoff = 10)

ggplot(overlap_diffs, aes(x=overlap_diffs,color=cancer)) + 
  geom_vline(aes(xintercept = 0), linetype = "dashed") +
  geom_density() + 
  ggtitle(paste0("# Pairs Below 0: PAAD - ", sum(possible_essential_pairs_10$overlap_diff < 0),
                 " BRCA - ", sum(brca_all_pairs_10$overlap_diff < 0))) +
  BerginskiRMisc::theme_berginski() 
```

![](process_PAAD_from_cBio_files/figure-gfm/comparison%20to%20BRCA%2010-1.png)<!-- -->

### Min 5% alteration

``` r
tic();
possible_essential_pairs_05 = gene_alteration_rates_paad %>% 
  filter(del_mut_rate >= 0.05) %>%
  
  #produce a list of all the gene permutations
  rename(Hugo_Symbol_1 = Hugo_Symbol) %>%
  mutate(Hugo_Symbol_2 = Hugo_Symbol_1) %>%
  expand(Hugo_Symbol_1,Hugo_Symbol_2) %>% 
  
  #remove the duplicated permutation sets
  filter(Hugo_Symbol_1 < Hugo_Symbol_2) %>%
  rename(gene_1 = Hugo_Symbol_1, gene_2 = Hugo_Symbol_2) %>%

  #this isn't strictly needed, but since PTEN is focus of the analysis, this
  #swaps the location of PTEN to gene_1 when PTEN is in the gene_2 slot and
  #makes looking at the downstream PTEN data easier
  mutate(gene_1_PTEN = ifelse(gene_2 == "PTEN", "PTEN", gene_1),
         gene_2_PTEN = ifelse(gene_2 == "PTEN", gene_1, gene_2)) %>%
  mutate(gene_1 = gene_1_PTEN, gene_2 = gene_2_PTEN) %>%
  select(-gene_1_PTEN, - gene_2_PTEN) %>%

  #determine the number of cases where alterations overlap
  mutate(overlap_count = future_map2_int(gene_1,gene_2,
                                         find_overlap_count_matrix,
                                         alteration_matrix = tcga_paad_altered_matrix,
                                         .progress=T)) %>%

  #merge back in the alteration rates for both gene_1 and gene_2
  left_join(gene_alteration_rates_paad %>% 
              rename(gene_1_mut_rate = del_mut_rate),by=c('gene_1' = 'Hugo_Symbol')) %>%
  left_join(gene_alteration_rates_paad %>% 
              rename(gene_2_mut_rate = del_mut_rate),by=c('gene_2' = 'Hugo_Symbol')) %>%

  #calculate the expected number of overlaps from the base rates
  mutate(expected_overlap_count = gene_1_mut_rate * gene_2_mut_rate * total_samples) %>%
  mutate(overlap_diff = overlap_count - expected_overlap_count) %>%
  mutate(overlap_p_value = future_pmap_dbl(list(overlap_count,total_samples,gene_1_mut_rate*gene_2_mut_rate),
                                           run_binom_test,
                                           .progress = T)) %>%
  mutate(overlap_p_value_adjust = p.adjust(overlap_p_value, method = "fdr")) %>%
  mutate_if(is.numeric, funs(signif(.,3))) %>%
  write_rds(here('results/PAAD/all_pairs_05.rds')) %>%
  identity()
toc();
```

    ## 1.409 sec elapsed

``` r
apparent_gene_essential_pairs = possible_essential_pairs_05 %>%
  filter(overlap_diff < 0, overlap_p_value_adjust <= 0.05) %>%
  write_rds(here('results/PAAD/essential_pairs_05.rds'))

apparent_gene_essential_pairs_PTEN = apparent_gene_essential_pairs %>%
  filter(gene_1 == "PTEN" | gene_2 == "PTEN") %>%
  write_rds(here('results/PAAD/PTEN_essential_pairs_05.rds'))
```

``` r
brca_all_pairs_05 = read_rds(here('results/all_pairs_05.rds'))

overlap_diffs = data.frame(cancer = c(rep("PAAD",length(possible_essential_pairs_05$overlap_count)),
                                      rep("BRCA", length(brca_all_pairs_05$overlap_count))),
                           overlap_diffs = c(possible_essential_pairs_05$overlap_diff,
                                             brca_all_pairs_05$overlap_diff)) %>%
  mutate(min_cutoff = 05)

ggplot(overlap_diffs, aes(x=overlap_diffs,color=cancer)) + 
  geom_vline(aes(xintercept = 0), linetype = "dashed") +
  geom_density() + 
  ggtitle(paste0("# Pairs Below 0: PAAD - ", sum(possible_essential_pairs_05$overlap_diff < 0),
                   " BRCA - ", sum(brca_all_pairs_05$overlap_diff < 0))) +
  BerginskiRMisc::theme_berginski() 
```

![](process_PAAD_from_cBio_files/figure-gfm/comparison%20to%20BRCA%2005-1.png)<!-- -->

### Min 1% alteration

``` r
tic();
possible_essential_pairs_01 = gene_alteration_rates_paad %>% 
  filter(del_mut_rate >= 0.01) %>%
  
  #produce a list of all the gene permutations
  rename(Hugo_Symbol_1 = Hugo_Symbol) %>%
  mutate(Hugo_Symbol_2 = Hugo_Symbol_1) %>%
  expand(Hugo_Symbol_1,Hugo_Symbol_2) %>% 
  
  #remove the duplicated permutation sets
  filter(Hugo_Symbol_1 < Hugo_Symbol_2) %>%
  rename(gene_1 = Hugo_Symbol_1, gene_2 = Hugo_Symbol_2) %>%

  #this isn't strictly needed, but since PTEN is focus of the analysis, this
  #swaps the location of PTEN to gene_1 when PTEN is in the gene_2 slot and
  #makes looking at the downstream PTEN data easier
  mutate(gene_1_PTEN = ifelse(gene_2 == "PTEN", "PTEN", gene_1),
         gene_2_PTEN = ifelse(gene_2 == "PTEN", gene_1, gene_2)) %>%
  mutate(gene_1 = gene_1_PTEN, gene_2 = gene_2_PTEN) %>%
  select(-gene_1_PTEN, - gene_2_PTEN) %>%

  #determine the number of cases where alterations overlap
  mutate(overlap_count = future_map2_int(gene_1,gene_2,
                                         find_overlap_count_matrix,
                                         alteration_matrix = tcga_paad_altered_matrix,
                                         .progress=T)) %>%
  

  #merge back in the alteration rates for both gene_1 and gene_2
  left_join(gene_alteration_rates_paad %>%
              rename(gene_1_mut_rate = del_mut_rate),by=c('gene_1' = 'Hugo_Symbol')) %>%
  left_join(gene_alteration_rates_paad %>%
              rename(gene_2_mut_rate = del_mut_rate),by=c('gene_2' = 'Hugo_Symbol')) %>%

  #calculate the expected number of overlaps from the base rates
  mutate(expected_overlap_count = gene_1_mut_rate * gene_2_mut_rate * total_samples) %>%
  mutate(overlap_diff = overlap_count - expected_overlap_count) %>%
  mutate(overlap_p_value = future_pmap_dbl(list(overlap_count,total_samples,gene_1_mut_rate*gene_2_mut_rate),
                                           run_binom_test,
                                           .progress = T)) %>%
  mutate(overlap_p_value_adjust = p.adjust(overlap_p_value, method = "fdr")) %>%
  write_rds(here('results/PAAD/all_pairs_01.rds')) %>%
  mutate_if(is.numeric, funs(signif(.,3))) %>%
  identity()
```

    ##  Progress: ──                                                                                                         100% Progress: ───                                                                                                        100% Progress: ────                                                                                                       100% Progress: ─────                                                                                                      100% Progress: ──────                                                                                                     100% Progress: ───────                                                                                                    100% Progress: ────────                                                                                                   100% Progress: ─────────                                                                                                  100% Progress: ─────────                                                                                                  100% Progress: ──────────                                                                                                 100% Progress: ───────────                                                                                                100% Progress: ────────────                                                                                               100% Progress: ─────────────                                                                                              100% Progress: ──────────────                                                                                             100% Progress: ───────────────                                                                                            100% Progress: ────────────────                                                                                           100% Progress: ─────────────────                                                                                          100% Progress: ─────────────────                                                                                          100% Progress: ──────────────────                                                                                         100% Progress: ───────────────────                                                                                        100% Progress: ───────────────────                                                                                        100% Progress: ────────────────────                                                                                       100% Progress: ─────────────────────                                                                                      100% Progress: ──────────────────────                                                                                     100% Progress: ──────────────────────                                                                                     100% Progress: ───────────────────────                                                                                    100% Progress: ────────────────────────                                                                                   100% Progress: ─────────────────────────                                                                                  100% Progress: ──────────────────────────                                                                                 100% Progress: ───────────────────────────                                                                                100% Progress: ───────────────────────────                                                                                100% Progress: ─────────────────────────────                                                                              100% Progress: ─────────────────────────────                                                                              100% Progress: ──────────────────────────────                                                                             100% Progress: ───────────────────────────────                                                                            100% Progress: ────────────────────────────────                                                                           100% Progress: ─────────────────────────────────                                                                          100% Progress: ─────────────────────────────────                                                                          100% Progress: ──────────────────────────────────                                                                         100% Progress: ───────────────────────────────────                                                                        100% Progress: ────────────────────────────────────                                                                       100% Progress: ─────────────────────────────────────                                                                      100% Progress: ──────────────────────────────────────                                                                     100% Progress: ──────────────────────────────────────                                                                     100% Progress: ───────────────────────────────────────                                                                    100% Progress: ────────────────────────────────────────                                                                   100% Progress: ─────────────────────────────────────────                                                                  100% Progress: ──────────────────────────────────────────                                                                 100% Progress: ───────────────────────────────────────────                                                                100% Progress: ───────────────────────────────────────────                                                                100% Progress: ────────────────────────────────────────────                                                               100% Progress: ─────────────────────────────────────────────                                                              100% Progress: ──────────────────────────────────────────────                                                             100% Progress: ───────────────────────────────────────────────                                                            100% Progress: ───────────────────────────────────────────────                                                            100% Progress: ────────────────────────────────────────────────                                                           100% Progress: ─────────────────────────────────────────────────                                                          100% Progress: ──────────────────────────────────────────────────                                                         100% Progress: ───────────────────────────────────────────────────                                                        100% Progress: ───────────────────────────────────────────────────                                                        100% Progress: ────────────────────────────────────────────────────                                                       100% Progress: ─────────────────────────────────────────────────────                                                      100% Progress: ──────────────────────────────────────────────────────                                                     100% Progress: ──────────────────────────────────────────────────────                                                     100% Progress: ───────────────────────────────────────────────────────                                                    100% Progress: ────────────────────────────────────────────────────────                                                   100% Progress: ─────────────────────────────────────────────────────────                                                  100% Progress: ──────────────────────────────────────────────────────────                                                 100% Progress: ───────────────────────────────────────────────────────────                                                100% Progress: ────────────────────────────────────────────────────────────                                               100% Progress: ─────────────────────────────────────────────────────────────                                              100% Progress: ──────────────────────────────────────────────────────────────                                             100% Progress: ───────────────────────────────────────────────────────────────                                            100% Progress: ────────────────────────────────────────────────────────────────                                           100% Progress: ─────────────────────────────────────────────────────────────────                                          100% Progress: ──────────────────────────────────────────────────────────────────                                         100% Progress: ───────────────────────────────────────────────────────────────────                                        100% Progress: ────────────────────────────────────────────────────────────────────                                       100% Progress: ─────────────────────────────────────────────────────────────────────                                      100% Progress: ──────────────────────────────────────────────────────────────────────                                     100% Progress: ───────────────────────────────────────────────────────────────────────                                    100% Progress: ────────────────────────────────────────────────────────────────────────                                   100% Progress: ────────────────────────────────────────────────────────────────────────                                   100% Progress: ─────────────────────────────────────────────────────────────────────────                                  100% Progress: ──────────────────────────────────────────────────────────────────────────                                 100% Progress: ───────────────────────────────────────────────────────────────────────────                                100% Progress: ────────────────────────────────────────────────────────────────────────────                               100% Progress: ─────────────────────────────────────────────────────────────────────────────                              100% Progress: ──────────────────────────────────────────────────────────────────────────────                             100% Progress: ───────────────────────────────────────────────────────────────────────────────                            100% Progress: ────────────────────────────────────────────────────────────────────────────────                           100% Progress: ────────────────────────────────────────────────────────────────────────────────                           100% Progress: ─────────────────────────────────────────────────────────────────────────────────                          100% Progress: ──────────────────────────────────────────────────────────────────────────────────                         100% Progress: ───────────────────────────────────────────────────────────────────────────────────                        100% Progress: ────────────────────────────────────────────────────────────────────────────────────                       100% Progress: ─────────────────────────────────────────────────────────────────────────────────────                      100% Progress: ──────────────────────────────────────────────────────────────────────────────────────                     100% Progress: ───────────────────────────────────────────────────────────────────────────────────────                    100% Progress: ───────────────────────────────────────────────────────────────────────────────────────                    100% Progress: ────────────────────────────────────────────────────────────────────────────────────────                   100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────                  100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────                 100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────                100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────               100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────              100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────             100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────────            100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────           100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────          100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────────         100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────────────        100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────       100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────      100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────────────     100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────────────     100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────────────────    100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────────────────    100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────────────────    100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────── 100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────── 100%
    ## 
    ##  Progress: ─                                                                                                          100% Progress: ─                                                                                                          100% Progress: ──                                                                                                         100% Progress: ──                                                                                                         100% Progress: ───                                                                                                        100% Progress: ────                                                                                                       100% Progress: ────                                                                                                       100% Progress: ─────                                                                                                      100% Progress: ─────                                                                                                      100% Progress: ──────                                                                                                     100% Progress: ───────                                                                                                    100% Progress: ───────                                                                                                    100% Progress: ────────                                                                                                   100% Progress: ────────                                                                                                   100% Progress: ─────────                                                                                                  100% Progress: ─────────                                                                                                  100% Progress: ──────────                                                                                                 100% Progress: ──────────                                                                                                 100% Progress: ───────────                                                                                                100% Progress: ────────────                                                                                               100% Progress: ────────────                                                                                               100% Progress: ─────────────                                                                                              100% Progress: ─────────────                                                                                              100% Progress: ──────────────                                                                                             100% Progress: ───────────────                                                                                            100% Progress: ───────────────                                                                                            100% Progress: ────────────────                                                                                           100% Progress: ────────────────                                                                                           100% Progress: ─────────────────                                                                                          100% Progress: ──────────────────                                                                                         100% Progress: ──────────────────                                                                                         100% Progress: ───────────────────                                                                                        100% Progress: ───────────────────                                                                                        100% Progress: ────────────────────                                                                                       100% Progress: ────────────────────                                                                                       100% Progress: ─────────────────────                                                                                      100% Progress: ─────────────────────                                                                                      100% Progress: ──────────────────────                                                                                     100% Progress: ──────────────────────                                                                                     100% Progress: ──────────────────────                                                                                     100% Progress: ───────────────────────                                                                                    100% Progress: ───────────────────────                                                                                    100% Progress: ───────────────────────                                                                                    100% Progress: ────────────────────────                                                                                   100% Progress: ────────────────────────                                                                                   100% Progress: ─────────────────────────                                                                                  100% Progress: ──────────────────────────                                                                                 100% Progress: ──────────────────────────                                                                                 100% Progress: ───────────────────────────                                                                                100% Progress: ───────────────────────────                                                                                100% Progress: ───────────────────────────                                                                                100% Progress: ────────────────────────────                                                                               100% Progress: ─────────────────────────────                                                                              100% Progress: ─────────────────────────────                                                                              100% Progress: ──────────────────────────────                                                                             100% Progress: ──────────────────────────────                                                                             100% Progress: ───────────────────────────────                                                                            100% Progress: ───────────────────────────────                                                                            100% Progress: ────────────────────────────────                                                                           100% Progress: ────────────────────────────────                                                                           100% Progress: ─────────────────────────────────                                                                          100% Progress: ─────────────────────────────────                                                                          100% Progress: ──────────────────────────────────                                                                         100% Progress: ──────────────────────────────────                                                                         100% Progress: ───────────────────────────────────                                                                        100% Progress: ───────────────────────────────────                                                                        100% Progress: ────────────────────────────────────                                                                       100% Progress: ─────────────────────────────────────                                                                      100% Progress: ─────────────────────────────────────                                                                      100% Progress: ──────────────────────────────────────                                                                     100% Progress: ──────────────────────────────────────                                                                     100% Progress: ───────────────────────────────────────                                                                    100% Progress: ───────────────────────────────────────                                                                    100% Progress: ────────────────────────────────────────                                                                   100% Progress: ─────────────────────────────────────────                                                                  100% Progress: ─────────────────────────────────────────                                                                  100% Progress: ──────────────────────────────────────────                                                                 100% Progress: ───────────────────────────────────────────                                                                100% Progress: ───────────────────────────────────────────                                                                100% Progress: ────────────────────────────────────────────                                                               100% Progress: ────────────────────────────────────────────                                                               100% Progress: ─────────────────────────────────────────────                                                              100% Progress: ──────────────────────────────────────────────                                                             100% Progress: ──────────────────────────────────────────────                                                             100% Progress: ───────────────────────────────────────────────                                                            100% Progress: ────────────────────────────────────────────────                                                           100% Progress: ────────────────────────────────────────────────                                                           100% Progress: ─────────────────────────────────────────────────                                                          100% Progress: ─────────────────────────────────────────────────                                                          100% Progress: ──────────────────────────────────────────────────                                                         100% Progress: ───────────────────────────────────────────────────                                                        100% Progress: ───────────────────────────────────────────────────                                                        100% Progress: ────────────────────────────────────────────────────                                                       100% Progress: ────────────────────────────────────────────────────                                                       100% Progress: ─────────────────────────────────────────────────────                                                      100% Progress: ─────────────────────────────────────────────────────                                                      100% Progress: ──────────────────────────────────────────────────────                                                     100% Progress: ───────────────────────────────────────────────────────                                                    100% Progress: ───────────────────────────────────────────────────────                                                    100% Progress: ────────────────────────────────────────────────────────                                                   100% Progress: ────────────────────────────────────────────────────────                                                   100% Progress: ─────────────────────────────────────────────────────────                                                  100% Progress: ─────────────────────────────────────────────────────────                                                  100% Progress: ──────────────────────────────────────────────────────────                                                 100% Progress: ───────────────────────────────────────────────────────────                                                100% Progress: ───────────────────────────────────────────────────────────                                                100% Progress: ────────────────────────────────────────────────────────────                                               100% Progress: ────────────────────────────────────────────────────────────                                               100% Progress: ─────────────────────────────────────────────────────────────                                              100% Progress: ─────────────────────────────────────────────────────────────                                              100% Progress: ──────────────────────────────────────────────────────────────                                             100% Progress: ───────────────────────────────────────────────────────────────                                            100% Progress: ───────────────────────────────────────────────────────────────                                            100% Progress: ────────────────────────────────────────────────────────────────                                           100% Progress: ─────────────────────────────────────────────────────────────────                                          100% Progress: ─────────────────────────────────────────────────────────────────                                          100% Progress: ──────────────────────────────────────────────────────────────────                                         100% Progress: ──────────────────────────────────────────────────────────────────                                         100% Progress: ───────────────────────────────────────────────────────────────────                                        100% Progress: ────────────────────────────────────────────────────────────────────                                       100% Progress: ────────────────────────────────────────────────────────────────────                                       100% Progress: ─────────────────────────────────────────────────────────────────────                                      100% Progress: ─────────────────────────────────────────────────────────────────────                                      100% Progress: ──────────────────────────────────────────────────────────────────────                                     100% Progress: ──────────────────────────────────────────────────────────────────────                                     100% Progress: ───────────────────────────────────────────────────────────────────────                                    100% Progress: ────────────────────────────────────────────────────────────────────────                                   100% Progress: ────────────────────────────────────────────────────────────────────────                                   100% Progress: ─────────────────────────────────────────────────────────────────────────                                  100% Progress: ─────────────────────────────────────────────────────────────────────────                                  100% Progress: ──────────────────────────────────────────────────────────────────────────                                 100% Progress: ───────────────────────────────────────────────────────────────────────────                                100% Progress: ───────────────────────────────────────────────────────────────────────────                                100% Progress: ────────────────────────────────────────────────────────────────────────────                               100% Progress: ─────────────────────────────────────────────────────────────────────────────                              100% Progress: ─────────────────────────────────────────────────────────────────────────────                              100% Progress: ──────────────────────────────────────────────────────────────────────────────                             100% Progress: ───────────────────────────────────────────────────────────────────────────────                            100% Progress: ───────────────────────────────────────────────────────────────────────────────                            100% Progress: ────────────────────────────────────────────────────────────────────────────────                           100% Progress: ─────────────────────────────────────────────────────────────────────────────────                          100% Progress: ─────────────────────────────────────────────────────────────────────────────────                          100% Progress: ──────────────────────────────────────────────────────────────────────────────────                         100% Progress: ───────────────────────────────────────────────────────────────────────────────────                        100% Progress: ───────────────────────────────────────────────────────────────────────────────────                        100% Progress: ────────────────────────────────────────────────────────────────────────────────────                       100% Progress: ─────────────────────────────────────────────────────────────────────────────────────                      100% Progress: ─────────────────────────────────────────────────────────────────────────────────────                      100% Progress: ──────────────────────────────────────────────────────────────────────────────────────                     100% Progress: ──────────────────────────────────────────────────────────────────────────────────────                     100% Progress: ───────────────────────────────────────────────────────────────────────────────────────                    100% Progress: ────────────────────────────────────────────────────────────────────────────────────────                   100% Progress: ────────────────────────────────────────────────────────────────────────────────────────                   100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────                  100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────                 100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────                 100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────                100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────               100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────               100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────              100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────             100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────             100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────────            100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────           100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────           100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────          100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────────         100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────────         100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────────────        100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────       100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────       100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────      100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────────────     100% Progress: ──────────────────────────────────────────────────────────────────────────────────────────────────────     100% Progress: ───────────────────────────────────────────────────────────────────────────────────────────────────────    100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────   100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ─────────────────────────────────────────────────────────────────────────────────────────────────────────  100% Progress: ────────────────────────────────────────────────────────────────────────────────────────────────────────── 100%

``` r
toc();
```

    ## 292.317 sec elapsed

``` r
apparent_gene_essential_pairs = possible_essential_pairs_01 %>%
  filter(overlap_diff < 0, overlap_p_value_adjust <= 0.01) %>%
  write_rds(here('results/PAAD/essential_pairs_01.rds'))

apparent_gene_essential_pairs_PTEN = apparent_gene_essential_pairs %>%
  filter(gene_1 == "PTEN" | gene_2 == "PTEN") %>%
  write_rds(here('results/PAAD/PTEN_essential_pairs_01.rds'))
```

``` r
ggplot(possible_essential_pairs_01, aes(x=overlap_diff)) + 
  geom_vline(aes(xintercept = 0), linetype = "dashed") +
  geom_density() + 
  ggtitle(paste0("# Pairs Below 0: PAAD - ", sum(possible_essential_pairs_01$overlap_diff < 0),
                 "/",signif(mean(possible_essential_pairs_01$overlap_diff < 0),3))) +
  BerginskiRMisc::theme_berginski() 
```

![](process_PAAD_from_cBio_files/figure-gfm/histogram%20of%20overlap%20diffs-1.png)<!-- -->
