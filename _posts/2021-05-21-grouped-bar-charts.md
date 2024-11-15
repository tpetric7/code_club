---
layout: post
title: "How to create a grouped bar chart in R with ggplot2's geom_col and position_dodge functions (CC107)"
blurb: "Dodging stacked bar charts"
author: "PD Schloss"
date: 2021-05-21 11:30
comments: false
youtube: wo6FSaz9AlE
---

<iframe style="margin: 0 auto;display:block;" width="560" height="315" src="https://www.youtube.com/embed/{{ page.youtube }}" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


## Code

This is where we started before the episode

```R
library(tidyverse)
library(readxl)
library(ggtext)
library(RColorBrewer)

metadata <- read_excel("raw_data/schubert.metadata.xlsx", na="NA") %>%
  select(sample_id, disease_stat) %>%
  drop_na(disease_stat)

otu_counts <- read_tsv("raw_data/schubert.subsample.shared") %>%
  select(Group, starts_with("Otu")) %>%
  rename(sample_id = Group) %>%
  pivot_longer(-sample_id, names_to="otu", values_to = "count")

taxonomy <- read_tsv("raw_data/schubert.cons.taxonomy") %>%
  select("OTU", "Taxonomy") %>%
  rename_all(tolower) %>%
  mutate(taxonomy = str_replace_all(taxonomy, "\\(\\d+\\)", ""),
         taxonomy = str_replace(taxonomy, ";$", "")) %>%
  separate(taxonomy,
           into=c("kingdom", "phylum", "class", "order", "family", "genus"),
           sep=";")

otu_rel_abund <- inner_join(metadata, otu_counts, by="sample_id") %>%
  inner_join(., taxonomy, by="otu") %>%
  group_by(sample_id) %>%
  mutate(rel_abund = count / sum(count)) %>%
  ungroup() %>%
  select(-count) %>%
  pivot_longer(c("kingdom", "phylum", "class", "order", "family", "genus", "otu"),
               names_to="level",
               values_to="taxon") %>%
  mutate(disease_stat = factor(disease_stat,
                               levels=c("NonDiarrhealControl",
                                        "DiarrhealControl",
                                        "Case")))


taxon_rel_abund <- otu_rel_abund %>%
  filter(level=="phylum") %>%
  group_by(disease_stat, sample_id, taxon) %>%
  summarize(rel_abund = sum(rel_abund), .groups="drop") %>%
  group_by(disease_stat, taxon) %>%
  summarize(mean_rel_abund = 100*mean(rel_abund), .groups="drop") %>%
  mutate(taxon = str_replace(taxon,
                             "(.*)_unclassified", "Unclassified *\\1*"),
         taxon = str_replace(taxon,
                             "^(\\S*)$", "*\\1*"))

taxon_pool <- taxon_rel_abund %>%
  group_by(taxon) %>%
  summarize(pool = max(mean_rel_abund) < 3,
            mean = mean(mean_rel_abund),
            .groups="drop")

inner_join(taxon_rel_abund, taxon_pool, by="taxon") %>%
  mutate(taxon = if_else(pool, "Other", taxon)) %>%
  group_by(disease_stat, taxon) %>%
  summarize(mean_rel_abund = sum(mean_rel_abund),
            mean = min(mean),
            .groups="drop") %>%
  mutate(taxon = factor(taxon),
         taxon = fct_reorder(taxon, mean, .desc=TRUE),
         taxon = fct_shift(taxon, n=1)) %>%
  ggplot(aes(x=disease_stat, y=mean_rel_abund, fill=taxon)) +
  geom_col() +
  scale_fill_manual(name=NULL,
                        breaks=c("*Bacteroidetes*", "*Firmicutes*",
                                 "*Proteobacteria*", "*Verrucomicrobia*",
                                 "Other"),
                        values = c(brewer.pal(4, "Dark2"), "gray")) +
  scale_x_discrete(breaks=c("NonDiarrhealControl",
                            "DiarrhealControl",
                            "Case"),
                   labels=c("Healthy",
                            "Diarrhea,<br>*C. difficile*<br>negative",
                            "Diarrhea,<br>*C. difficile*<br>positive")) +
  scale_y_continuous(expand=c(0, 0)) +
  labs(x=NULL,
       y="Mean Relative Abundance (%)") +
  theme_classic() +
  theme(axis.text.x = element_markdown(),
        legend.text = element_markdown(),
        legend.key.size = unit(10, "pt"))

ggsave("schubert_stacked_bar.tiff", width=5, height=4)
```


## Installations

* [See video](https://www.youtube.com/watch?v=D6CunpqF04E) for instructions on getting set up
* [R](https://r-project.org)
* [RStudio](https://rstudio.com)
* [Raw data](https://github.com/riffomonas/raw_data/releases/latest)
