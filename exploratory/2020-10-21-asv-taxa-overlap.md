Quantifying the overlap of ASVs between taxa
================
Pat Schloss
10/21/2020

    library(tidyverse)
    library(here)
    library(knitr)

    metadata <- read_tsv(here("data/references/genome_id_taxonomy.tsv"),
                                             col_types = cols(.default = col_character())) %>%
        mutate(strain = if_else(scientific_name == species,
                                                        NA_character_,
                                                        scientific_name)) %>%
        select(-scientific_name)

    asv <- read_tsv(here("data/processed/rrnDB.count_tibble"),
                                    col_types = cols(.default = col_character(),
                                                                     count = col_integer()))

    metadata_asv <- inner_join(metadata, asv, by=c("genome_id" = "genome"))

### How often is the same ASV found in multiple taxa from the same rank?

Across the analyses, we’ve already seen that a single bacterial genome
can have multiple ASVs and that depending on the species we are looking
at, there may be as many ASVs as there are genome sequences for that
species. To me this says that ASVs pose a risk at unnaturally splitting
a species and even a genome into multiple taxonomic groupings! In my
book, that’s not good.

Today, I want to ask an equally important question: If I have an ASV,
what’s the probability that it is also found in another taxonomic group
from the same rank? In other words, if I have an ASV from *Bacillus
subtilis*, what’s the probability of finding it in *Bacillus cereus*? Of
course, it’s probably more likely to find a *Bacillus subtilis* ASV in a
more closely related organism like *Bacillus cereus* than *E. coli*.
Perhaps we can control for relatedness in a future episode, but for
today, I want to answer this question for any two taxa from the same
rank.

    # metadata_asv - input data
    overlap_data <- metadata_asv %>%
    # - focus on taxonomic ranks - kingdom to species, asv, and region
        select(-genome_id, -count, -strain) %>%
    # - make data frame tidy
        pivot_longer(cols=c(-region, -asv), names_to="rank", values_to="taxon") %>%
    # - remove lines from the data where we don't have a taxonomy
        drop_na(taxon) %>%
    # - remove redundant lines 
        distinct() %>%
    # for each region and taxonomic rank, group by asvs
        group_by(region, rank, asv) %>%
    # - for each asv - count the number of taxa
        summarize(n_taxa = n(), .groups="drop_last") %>%
    # - count the number of asvs that appear in more than one taxa
        count(n_taxa) %>%
    # - find the ratio of the number of asvs in more than one taxa to the total number of asvs
        summarize(overlap = sum((n_taxa > 1) * n) / sum(n), .groups="drop") %>%
        mutate(rank = factor(rank, levels=c("kingdom", "phylum", "class", "order", "family", "genus", "species"))) %>%
        mutate(overlap = 100*overlap)

    # create a plot showing specificity at each taxonomic rank for each region
    # x = taxonomic rank
    # y = specificity or % of asvs found in more than one taxa
    # geom = line plot
    # different lines for each region of 16S rRNA gene
    overlap_data %>%
        ggplot(aes(x=rank, y=overlap, group=region, color=region)) +
        geom_line()

![](2020-10-21-asv-taxa-overlap_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

    overlap_data %>%
        filter(rank == "species") %>% kable

| region | rank    |   overlap |
|:-------|:--------|----------:|
| v19    | species |  3.599863 |
| v34    | species |  8.759673 |
| v4     | species | 12.564060 |
| v45    | species |  9.927921 |

### Conclusions

-   Sub regions are less specific at the species level than full-length.
    Still full-length has 3.6% overlap whereas the subregions are
    between 8.8 and 12.6% overlap
-   This analysis does not control for uneven sampling of different
    species