# Dementia Gap Map

Interactive map of dementia and Alzheimer research that links literature dynamics to genetic, functional, pathway, drug, and clinical trial evidence.

The first prototype should answer a focused question:

> Which genetically supported dementia mechanisms are emerging in the literature, and have they translated into drugs or clinical trials yet?

This README is a handoff spec for a two-day prototype build. It combines the discussion so far, the NIH predictive-breakthroughs paper/repo approach, the GWAS/eQTL papers discussed, and concrete API routes for data collection.

## One Sentence Product

A map of papers discussing dementia, Alzheimer disease, GWAS loci, eQTLs, genes, pathways, drugs/interventions, and clinical trials, grouped by citation/co-citation similarity. When a user selects a topic group, they get a ranked view of the GWAS loci, genes, pathways, drugs/interventions, clinical trials, and evidence trajectory behind that topic.

## Core Decisions From The Discussion

Use a constrained build. Do not attempt to ingest all Alzheimer literature or reproduce the full NIH all-PubMed pipeline.

Use two linked layers:

```text
Layer A: literature/topic dynamics
papers -> citations/co-citations -> topic clusters -> topic trajectories over time

Layer B: biological/translational evidence
GWAS/eQTL -> variant/locus -> gene/target -> pathway -> drug/intervention -> clinical trial
```

The bridge is:

```text
topic cluster -> member papers -> GWAS studies / variants / genes / pathways -> target/trial map
```

The practical demo claim should be:

```text
We can map dementia research topics over time and show which genetically supported areas are emerging, saturated, or under-translated into trials.
```

Avoid claiming "we predict breakthroughs" in the prototype. Say "we detect emerging, genetically supported areas and compare them with clinical translation."

## What We Are Borrowing From Predictive Breakthroughs

Paper: `Prediction of transformative breakthroughs in biomedical research`, bioRxiv DOI `10.64898/2025.12.16.694385`.

GitHub: https://github.com/NIHOPA/predictivebreakthroughs

Important caveat: the repo README says the full all-PubMed co-citation/RMCL run requires hundreds of GB of RAM and billions of computations. Also, the repo states that use of the software requires a license. Treat it as a reference design, not code to copy.

Ideas to borrow:

- Build a paper network, not just a keyword search.
- Cluster papers into topics.
- Link clusters across years to create topic trajectories.
- Score topics using emergence-like features:
  - burst of recent papers
  - growth of ancestor topic
  - low early cohesion or high mixedness
  - influential papers
- Prefer article-level metrics over journal-level metrics.
- Track topic split/merge behavior.

Prototype adaptation:

```text
NIH method:
all PubMed -> co-citation network -> RMCL topics -> yearly trajectories -> logistic regression signal

Our two-day method:
AD/dementia genetics corpus -> citation/co-citation or bibliographic coupling network -> Leiden/Louvain topics -> yearly summaries -> transparent heuristic scores
```

## What The Two Nature Genetics Papers Add

### 1. AD/ADRD GWAS Consensus Meta-analysis

URL: https://www.nature.com/articles/s41588-026-02583-1

Use this as the first genetics backbone.

Role in the map:

```text
GWAS disease association layer:
locus / variant / mapped gene / publication / trait / evidence strength
```

It gives the "genetic evidence floor" for the prototype. The question for each locus/gene is: does this genetically supported area have functional evidence and clinical translation?

### 2. SingleBrain Single-nucleus eQTL Meta-analysis

URL: https://www.nature.com/articles/s41588-026-02541-x

Use this as the first functional interpretation layer.

Role in the map:

```text
GWAS locus -> regulatory variant -> brain cell type eQTL -> candidate causal gene -> mechanism
```

This is especially important because many GWAS variants are noncoding. GWAS tells us a region is associated with disease; eQTL/colocalization helps identify which gene and cell type may mediate the signal.

## Why Co-citation Links To GWAS Loci And Trials

Co-citation or citation similarity does not directly say "this variant causes disease." It says "these papers are treated by the field as intellectually related."

GWAS/eQTL and trial data provide the biological and translational meaning.

Example:

```text
Topic cluster:
microglial lipid handling in Alzheimer disease

Member papers:
AD GWAS papers, TREM2/PLCG2/ABCA7 papers, microglia single-cell papers

Attached evidence:
GWAS loci: APOE, TREM2, PLCG2, ABI3, ABCA7, INPP5D
Cell type: microglia
Pathways: lipid metabolism, innate immunity, phagocytosis
Trials: low or moderate direct activity

Interpretation:
high genetic support + emerging literature + limited clinical translation
```

The paper network is the field dynamics layer. The GWAS/eQTL/trial graph is the evidence layer.

## Minimal Viable Build

For two days, build a static-data prototype with reproducible ingestion scripts.

Target scale:

- 300 to 1,000 papers
- 100 to 300 GWAS studies/associations
- 50 to 150 high-priority genes/targets
- all public Alzheimer disease ClinicalTrials.gov records, then filter to therapeutic or biomarker studies
- 10 to 25 visible topic clusters

The UI should be useful even if the data is incomplete. Every score must be explainable.

## Data Sources

### GWAS Catalog

Use for curated GWAS studies, disease traits, variants, p-values, reported genes, Ensembl IDs, PMIDs, ancestry, sample size, and study accessions.

Docs:

- https://www.ebi.ac.uk/gwas/docs/api
- https://www.ebi.ac.uk/gwas/rest/api

Verified useful endpoints:

```http
GET https://www.ebi.ac.uk/gwas/rest/api/studies/search/findByEfoTrait?efoTrait=Alzheimer%20disease&page=0&size=100
```

This is the safest first GWAS seed. On 2026-07-01 it returned paginated study rows for Alzheimer disease.

Important response fields:

```text
accessionId
diseaseTrait.trait
publicationInfo.pubmedId
publicationInfo.publicationDate
publicationInfo.title
publicationInfo.publication
publicationInfo.author.fullname
initialSampleSize
replicationSampleSize
snpCount
ancestries
_links.associations.href
_links.associationsByStudySummary.href
```

For each study:

```http
GET https://www.ebi.ac.uk/gwas/rest/api/studies/{ACCESSION_ID}/associations
```

or:

```http
GET https://www.ebi.ac.uk/gwas/rest/api/studies/{ACCESSION_ID}/associations?projection=associationByStudy
```

Association search by EFO trait also works:

```http
GET https://www.ebi.ac.uk/gwas/rest/api/associations/search/findByEfoTrait?efoTrait=Alzheimer%20disease
```

Caveat: the association EFO endpoint can return a large unpaginated payload. Cache it once and normalize locally.

Useful association fields:

```text
pvalue
pvalueMantissa
pvalueExponent
orPerCopyNum
betaNum
betaDirection
riskFrequency
snpType
loci[].strongestRiskAlleles[].riskAlleleName
loci[].authorReportedGenes[].geneName
loci[].authorReportedGenes[].entrezGeneIds
loci[].authorReportedGenes[].ensemblGeneIds
_links.study.href
_links.snps.href
_links.efoTraits.href
```

Recommended GWAS ingestion:

1. Query studies by EFO trait.
2. Page through all results.
3. Store `gwas_studies`.
4. For each `accessionId`, fetch associations.
5. Store `gwas_associations`.
6. Extract all PMIDs from `publicationInfo.pubmedId`.
7. Use those PMIDs as seed papers for the literature network.

### Open Targets Platform

Use for target-disease association scores, genetic association scores, target metadata, tractability, disease IDs, GWAS/molecular QTL studies, credible sets, and L2G/colocalization expansion where available.

Docs:

- https://platform-docs.opentargets.org/data-access/graphql-api
- GraphQL endpoint: https://api.platform.opentargets.org/api/v4/graphql

Verified Alzheimer disease ID:

```text
MONDO_0004975 = Alzheimer disease
```

Search query:

```graphql
query search($queryString: String!) {
  search(queryString: $queryString, entityNames: ["disease"], page: { index: 0, size: 5 }) {
    hits {
      id
      name
      entity
    }
  }
}
```

Variables:

```json
{ "queryString": "Alzheimer disease" }
```

Associated targets query:

```graphql
query diseaseTargets($efoId: String!) {
  disease(efoId: $efoId) {
    id
    name
    associatedTargets(page: { index: 0, size: 100 }) {
      rows {
        target {
          id
          approvedSymbol
          approvedName
        }
        score
        datatypeScores {
          id
          score
        }
      }
    }
  }
}
```

Variables:

```json
{ "efoId": "MONDO_0004975" }
```

Useful `datatypeScores`:

```text
genetic_association
genetic_literature
literature
clinical
affected_pathway
rna_expression
animal_model
```

GWAS/molecular QTL studies by disease:

```graphql
query studies($diseaseIds: [String!]) {
  studies(diseaseIds: $diseaseIds, enableIndirect: true, page: { index: 0, size: 100 }) {
    count
    rows {
      id
      studyType
      traitFromSource
      publicationFirstAuthor
      publicationDate
      publicationJournal
      nCases
    }
  }
}
```

Variables:

```json
{ "diseaseIds": ["MONDO_0004975"] }
```

Caveat: the current GraphQL schema has changed from some older examples. Drug fields were not available on the `Disease` or `Target` objects in the quick test. For drugs/trials in the two-day build, use ClinicalTrials.gov directly and optionally add Open Targets downloadable datasets later.

### ClinicalTrials.gov

Use for clinical translation and saturation.

Docs:

- https://clinicaltrials.gov/data-api/api

Verified endpoint:

```http
GET https://clinicaltrials.gov/api/v2/studies?query.cond=Alzheimer%20Disease&pageSize=100&format=json
```

Pagination:

```text
response.nextPageToken -> use pageToken={nextPageToken}
```

Important response fields:

```text
protocolSection.identificationModule.nctId
protocolSection.identificationModule.briefTitle
protocolSection.identificationModule.officialTitle
protocolSection.statusModule.overallStatus
protocolSection.statusModule.startDateStruct.date
protocolSection.statusModule.primaryCompletionDateStruct.date
protocolSection.statusModule.completionDateStruct.date
protocolSection.sponsorCollaboratorsModule.leadSponsor.name
protocolSection.sponsorCollaboratorsModule.leadSponsor.class
protocolSection.conditionsModule.conditions
protocolSection.conditionsModule.keywords
protocolSection.designModule.studyType
protocolSection.designModule.phases
protocolSection.designModule.enrollmentInfo.count
protocolSection.armsInterventionsModule.interventions
protocolSection.outcomesModule.primaryOutcomes
derivedSection.conditionBrowseModule.meshes
hasResults
```

Processing:

1. Pull all Alzheimer disease studies.
2. Normalize into `trials`.
3. Extract intervention names from `armsInterventionsModule.interventions`.
4. Classify trial type:
   - therapeutic
   - diagnostic/biomarker
   - imaging
   - observational
   - lifestyle/care
5. For MVP, manually map common interventions/mechanisms to pathways:
   - amyloid
   - tau
   - cholinergic/symptomatic
   - inflammation/microglia
   - lipid/metabolism
   - vascular
   - synaptic/neuroprotection
   - diagnostic biomarker
6. Later, replace manual mapping with ChEMBL/Open Targets/DrugBank-style target mapping.

### PubMed / NCBI E-utilities

Use for seed expansion, references, cited-by lists, metadata, and year-by-year literature trajectories.

Docs:

- https://www.ncbi.nlm.nih.gov/books/NBK25501/
- Base: https://eutils.ncbi.nlm.nih.gov/entrez/eutils/

Search example:

```http
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=(Alzheimer%20Disease[MeSH%20Terms]%20OR%20Alzheimer*[Title/Abstract])%20AND%20(GWAS%20OR%20genome-wide%20association%20OR%20genetics%20OR%20eQTL)&retmode=json&retmax=500
```

Metadata:

```http
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi?db=pubmed&id=35379992,30514930&retmode=json
```

References for a PMID:

```http
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/elink.fcgi?dbfrom=pubmed&db=pubmed&id=35379992&linkname=pubmed_pubmed_refs&retmode=json
```

Cited-by for a PMID:

```http
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/elink.fcgi?dbfrom=pubmed&db=pubmed&id=35379992&linkname=pubmed_pubmed_citedin&retmode=json
```

Use cases:

- `pubmed_pubmed_refs` supports bibliographic coupling.
- `pubmed_pubmed_citedin` supports co-citation by shared citing papers.
- `esummary` or `efetch` gives title, journal, year, authors, publication types.

Caveats:

- Cited-by lists can be very large for major papers.
- Use request batching and local caching.
- For strict historical analysis, do not count citations from papers published after the year being analyzed.
- For the two-day prototype, a current-state co-citation network is acceptable if the UI labels it clearly as current-state.

### NIH iCite

Use for article-level citation influence and translational metrics.

Docs:

- https://icite.od.nih.gov/api

Verified minimal query:

```http
GET https://icite.od.nih.gov/api/pubs?pmids=35379992&fl=pmid,title,year,relative_citation_ratio,citation_count,apt,is_clinical
```

Useful fields:

```text
pmid
title
year
relative_citation_ratio
citation_count
apt
is_clinical
```

Use in scores:

- `relative_citation_ratio` -> influence
- `citation_count` -> attention
- `apt` -> approximate potential to translate
- `is_clinical` -> clinical orientation of paper/topic

Caveat: current RCR and citation counts include future information if used for historical prediction. For the MVP, use them as current-state influence metrics, not retrospective predictions.

### Europe PMC

Use as an optional alternative for publication metadata, abstracts, citations, grants, and text-mined annotations.

Docs:

- https://europepmc.org/RestfulWebService

Useful search pattern:

```http
GET https://www.ebi.ac.uk/europepmc/webservices/rest/search?query=TITLE_ABS:%22Alzheimer%22%20AND%20GWAS&format=json&pageSize=100
```

Use if PubMed metadata is insufficient or if text-mined gene/disease annotations are needed.

### Semantic Scholar / OpenAlex

Use as optional fallbacks for citation metadata, references, citation counts, and broader paper coverage.

Semantic Scholar docs:

- https://api.semanticscholar.org/api-docs/graph

OpenAlex docs:

- https://developers.openalex.org/

Notes:

- OpenAlex search was rate-limited anonymously during testing. Use an API key if relying on it.
- Semantic Scholar may not resolve every PMID directly. Use DOI or title fallback if PMID lookup fails.
- For this prototype, PubMed ELink plus iCite is the more reliable default.

### Functional eQTL / Colocalization Data

Use the SingleBrain paper as the first functional layer.

For the two-day prototype, do one of these:

1. Manual CSV from paper tables/supplement:

```text
variant_or_locus
gene_symbol
ensembl_gene_id
cell_type
evidence_type
colocalization_score
paper
pmid_or_doi
```

2. Open Targets credible sets / studies expansion:

Use Open Targets `studies`, `credibleSet`, and `credibleSets` queries after the MVP if the schema is convenient.

3. If no functional file is ready:

Use Open Targets target-disease `datatypeScores` as the temporary functional evidence proxy:

```text
genetic_association
rna_expression
affected_pathway
```

### Pathway Mapping

For the two-day prototype, use a curated pathway mapping file rather than trying to solve pathway annotation fully.

Example:

```csv
gene_symbol,pathway_group,notes
APP,amyloid,amyloid precursor protein
PSEN1,amyloid,gamma-secretase
PSEN2,amyloid,gamma-secretase
MAPT,tau,tau pathology
APOE,lipid_metabolism,lipoprotein biology
TREM2,microglia_immune,microglial receptor
PLCG2,microglia_immune,immune signaling
ABI3,microglia_immune,immune cytoskeletal regulation
ABCA7,lipid_metabolism,lipid transport/phagocytosis
BIN1,endocytosis,endosomal trafficking
PICALM,endocytosis,clathrin-mediated endocytosis
CLU,lipid_metabolism,chaperone/lipid biology
SORL1,endosomal_lysosomal,endosomal trafficking
INPP5D,microglia_immune,immune signaling
CD33,microglia_immune,myeloid receptor
```

Later options:

- Reactome Content Service: https://reactome.org/ContentService/
- g:Profiler: https://biit.cs.ut.ee/gprofiler/page/apis
- GO annotations
- Open Targets target annotations

## Proposed Data Model

Use SQLite, DuckDB, or local Parquet/JSON for the first prototype. Do not start with Neo4j unless the team specifically needs graph queries. A table model is easier for a two-day build and can be converted to graph later.

### `papers`

```text
paper_id             internal stable ID
pmid                 PubMed ID, nullable
doi                  DOI, nullable
title
abstract
year
journal
authors
source               gwas_catalog | pubmed_search | cited_by_expansion | reference_expansion | manual_seed
is_seed              boolean
is_gwas_study        boolean
is_eqtl_study        boolean
is_clinical          from iCite if available
relative_citation_ratio
citation_count
apt
url
```

### `paper_references`

```text
paper_id
referenced_paper_id
referenced_pmid
source
```

### `paper_citations`

```text
paper_id
citing_paper_id
citing_pmid
citing_year
source
```

### `paper_edges`

```text
source_paper_id
target_paper_id
edge_type             bibliographic_coupling | co_citation | text_similarity
weight
year_snapshot         nullable for static network
shared_count
source
```

### `topic_years`

```text
topic_year_id
topic_id
year
label
paper_count
new_paper_count_2y
pct_new
mean_rcr
sum_rcr
mean_apt
clinical_paper_fraction
entropy_gene
entropy_pathway
emergence_score
genetic_support_score
functional_support_score
clinical_translation_score
clinical_saturation_score
opportunity_score
```

### `topic_members`

```text
topic_year_id
paper_id
membership_weight
```

### `topic_links`

```text
from_topic_year_id
to_topic_year_id
link_type             progression | split | merge | weak_overlap | de_novo
jaccard_overlap
paper_overlap_count
```

### `gwas_studies`

```text
accession_id
pmid
title
journal
publication_date
first_author
disease_trait
initial_sample_size
replication_sample_size
snp_count
ancestry_summary
source_url
```

### `gwas_associations`

```text
association_id
accession_id
pmid
trait
variant_id            rsID or chr:pos allele
risk_allele
pvalue
or_per_copy
beta
beta_direction
reported_gene_symbol
ensembl_gene_id
entrez_gene_id
risk_frequency
snp_type
source_url
```

### `genes`

```text
gene_symbol
ensembl_gene_id
entrez_gene_id
approved_name
pathway_group
open_targets_score
open_targets_genetic_score
open_targets_clinical_score
open_targets_literature_score
tractability_summary
```

### `functional_links`

```text
variant_or_locus
gene_symbol
ensembl_gene_id
cell_type
evidence_type          eQTL | colocalization | L2G | expression | manual
score
source_paper
source_url
```

### `trials`

```text
nct_id
brief_title
official_title
overall_status
phase
study_type
start_date
primary_completion_date
completion_date
enrollment
lead_sponsor
lead_sponsor_class
conditions
interventions
has_results
trial_category         therapeutic | diagnostic | biomarker | observational | lifestyle | other
mechanism_group        amyloid | tau | inflammation | lipid | vascular | diagnostic | unknown
url
```

### `trial_targets`

```text
nct_id
intervention_name
gene_symbol
pathway_group
mapping_source         manual | opentargets | chembl | llm_reviewed
confidence
```

### `evidence_edges`

General graph-like edge table for the UI.

```text
source_type            paper | topic | variant | gene | pathway | drug | trial
source_id
target_type
target_id
edge_type
score
evidence_type
source_name
source_url
provenance_text
```

## Ingestion Pipeline

Recommended directory layout:

```text
data/
  raw/
    gwas_catalog/
    opentargets/
    pubmed/
    icite/
    clinical_trials/
    manual/
  processed/
    papers.parquet
    paper_edges.parquet
    topic_years.parquet
    topic_members.parquet
    gwas_associations.parquet
    genes.parquet
    trials.parquet
    evidence_edges.parquet
  app/
    map_data.json
    topic_details.json

scripts/
  ingest_gwas_catalog.py
  ingest_pubmed.py
  ingest_icite.py
  ingest_opentargets.py
  ingest_clinical_trials.py
  build_paper_network.py
  cluster_topics.py
  link_topics_to_evidence.py
  score_topics.py
  export_app_data.py
```

### Step 1: Seed GWAS Studies

Pseudo-code:

```python
def fetch_gwas_studies(efo_trait="Alzheimer disease"):
    page = 0
    rows = []
    while True:
        url = "https://www.ebi.ac.uk/gwas/rest/api/studies/search/findByEfoTrait"
        params = {"efoTrait": efo_trait, "page": page, "size": 100}
        data = get_json(url, params=params)
        rows.extend(data.get("_embedded", {}).get("studies", []))
        if page >= data["page"]["totalPages"] - 1:
            break
        page += 1
    return normalize_studies(rows)
```

Normalize PMIDs and use them as seed papers.

### Step 2: Fetch GWAS Associations

Pseudo-code:

```python
def fetch_associations_for_study(accession_id):
    url = f"https://www.ebi.ac.uk/gwas/rest/api/studies/{accession_id}/associations"
    data = get_json(url)
    return normalize_associations(data)
```

Extract:

- strongest risk allele
- rsID
- p-value
- odds ratio / beta
- reported genes
- Ensembl IDs
- study accession
- PMID

### Step 3: Add Manual Seed Papers

Start with:

```text
Nature Genetics 2026 AD/ADRD GWAS consensus paper
Nature Genetics 2026 SingleBrain eQTL paper
Bellenguez et al. 2022 AD/ADRD genetics paper, PMID 35379992
major AD GWAS papers surfaced by GWAS Catalog
```

Manual seed table:

```csv
pmid,doi,title,role
35379992,10.1038/s41588-022-01024-z,New insights into the genetic etiology of Alzheimer's disease and related dementias,gwas_backbone
```

For Nature 2026 papers, add DOI first if PMIDs are not yet available.

### Step 4: Expand Paper Corpus

For each seed PMID:

1. Fetch references via `pubmed_pubmed_refs`.
2. Fetch cited-by via `pubmed_pubmed_citedin`.
3. Keep expansion papers only if they pass one of:
   - title/abstract contains Alzheimer/dementia/genetics/GWAS/eQTL/single-cell/microglia/tau/amyloid
   - cites at least two seed papers
   - is cited by at least one seed paper and also appears in another seed neighborhood
   - is a GWAS Catalog publication

Hard cap:

```text
max papers: 1000
max cited-by per seed: 200 newest or highest RCR if needed
```

### Step 5: Fetch iCite Metrics

Batch PMIDs:

```http
GET https://icite.od.nih.gov/api/pubs?pmids={comma_separated_pmids}&fl=pmid,title,year,relative_citation_ratio,citation_count,apt,is_clinical
```

Store in `papers`.

### Step 6: Fetch Open Targets Associated Targets

Use `MONDO_0004975`.

Store:

- target Ensembl ID
- approved symbol
- overall association score
- datatype scores

Join to GWAS reported genes by Ensembl ID first, gene symbol second.

### Step 7: Fetch Clinical Trials

Loop pages:

```python
def fetch_trials():
    token = None
    rows = []
    while True:
        params = {
            "query.cond": "Alzheimer Disease",
            "pageSize": 100,
            "format": "json",
        }
        if token:
            params["pageToken"] = token
        data = get_json("https://clinicaltrials.gov/api/v2/studies", params=params)
        rows.extend(data.get("studies", []))
        token = data.get("nextPageToken")
        if not token:
            break
    return normalize_trials(rows)
```

Normalize interventions and classify mechanisms.

## Building The Paper Network

Use two edge types.

### Bibliographic Coupling

Two papers are linked if they cite the same references.

This is future-safe because references are known at publication time.

Formula:

```text
weight(A, B) = shared_references(A, B) / sqrt(reference_count(A) * reference_count(B))
```

Use this for historical topic snapshots if you want to avoid future leakage.

### Co-citation

Two papers are linked if later papers cite both of them.

Formula:

```text
weight(A, B) = shared_citers(A, B) / sqrt(citer_count(A) * citer_count(B))
```

Use this for current-state field structure.

For historical snapshots, only count citing papers with `citing_year <= snapshot_year`.

### Practical MVP Edge Strategy

For day one:

```text
network_weight = 0.7 * bibliographic_coupling + 0.3 * current_co_citation
```

If historical leakage is a concern:

```text
network_weight_year_Y = bibliographic_coupling among papers published <= Y
```

Then compute co-citation as a separate current-state overlay, not the cluster driver.

### Pseudo-code

```python
from collections import defaultdict
from itertools import combinations
from math import sqrt

def build_bibliographic_edges(paper_refs, min_weight=0.05):
    # paper_refs: dict[paper_id] -> set[referenced_paper_id]
    ref_to_papers = defaultdict(set)
    for paper_id, refs in paper_refs.items():
        for ref in refs:
            ref_to_papers[ref].add(paper_id)

    shared = defaultdict(int)
    for papers in ref_to_papers.values():
        for a, b in combinations(sorted(papers), 2):
            shared[(a, b)] += 1

    edges = []
    for (a, b), count in shared.items():
        denom = sqrt(len(paper_refs[a]) * len(paper_refs[b]))
        weight = count / denom if denom else 0
        if weight >= min_weight:
            edges.append((a, b, "bibliographic_coupling", weight, count))
    return edges

def build_cocitation_edges(paper_citers, min_weight=0.02):
    # paper_citers: dict[paper_id] -> set[citing_paper_id]
    papers = sorted(paper_citers)
    edges = []
    for a, b in combinations(papers, 2):
        shared = paper_citers[a] & paper_citers[b]
        if not shared:
            continue
        denom = sqrt(len(paper_citers[a]) * len(paper_citers[b]))
        weight = len(shared) / denom if denom else 0
        if weight >= min_weight:
            edges.append((a, b, "co_citation", weight, len(shared)))
    return edges
```

## Clustering Topics

For the prototype, use Leiden or Louvain via Python.

Suggested libraries:

```text
networkx
igraph
leidenalg
python-louvain
pandas
duckdb
scikit-learn
```

If installing graph packages is a blocker, use NetworkX greedy modularity or scikit-learn clustering on paper embeddings as a fallback.

Clustering output:

```text
paper_id -> topic_id
topic_id -> top papers, top terms, top genes, top pathways
```

Topic labels:

Generate deterministic labels from top terms and genes:

```text
microglia / TREM2 / immune signaling
amyloid / APP / secretase
lipid metabolism / APOE / ABCA7
endosomal trafficking / BIN1 / PICALM / SORL1
tau / MAPT / neurofibrillary pathology
vascular dementia / stroke / cardiovascular pleiotropy
biomarkers / plasma / CSF / imaging
```

For the demo, allow manual override labels in:

```text
data/manual/topic_labels.csv
```

## Topic Trajectories Over Time

MVP trajectory method:

1. Assign each paper to a topic using the full current network.
2. For each year, summarize papers in that topic with `paper.year <= year`.
3. Track counts and evidence growth by year.

Better method:

1. Build a network snapshot for each year.
2. Cluster each year separately.
3. Link yearly clusters by paper overlap.

Cluster linking:

```text
jaccard(A_year, B_year_plus_1) = |papers in both| / |papers in union|
```

Link rules:

```text
progression: one strong next-year match
split: one topic maps to two or more next-year topics
merge: two or more prior topics map to one next-year topic
de_novo: no meaningful prior topic
stagnant: low new paper growth
```

MVP recommendation:

Use the simpler full-network topic assignment and yearly summaries first. Add split/merge if time allows.

## Linking Topics To GWAS, Genes, Pathways, And Trials

### Topic -> GWAS

Join through paper PMID:

```text
topic_members.paper_id -> papers.pmid -> gwas_studies.pmid -> gwas_associations.accession_id
```

If a topic contains a GWAS Catalog study paper, attach all associations from that study to the topic.

### Topic -> Gene

Sources:

1. GWAS Catalog reported genes
2. Open Targets target-disease associations
3. eQTL/colocalization table
4. Simple title/abstract gene mentions as low-confidence evidence

Join priority:

```text
Ensembl ID > Entrez ID > HGNC gene symbol
```

### Gene -> Pathway

First prototype:

```text
manual curated gene_pathway.csv
```

Later:

```text
Reactome / GO / g:Profiler enrichment
```

### Pathway/Gene -> Trial

First prototype:

```text
manual intervention -> mechanism_group mapping
```

Examples:

```text
aducanumab, lecanemab, donanemab -> amyloid
semaglutide, insulin/metabolic interventions -> metabolism
anti-inflammatory or immune modulators -> inflammation/microglia
tau vaccines/anti-tau antibodies -> tau
plasma/CSF/PET diagnostic studies -> biomarker/diagnostic
```

Later:

```text
intervention -> ChEMBL/Open Targets drug -> target -> gene/pathway
```

## Scores

All scores should be normalized to 0 to 1 before combining.

### Genetic Support

Per gene/topic:

```text
genetic_support =
  0.35 * normalized(-log10(best_pvalue))
  + 0.25 * independent_gwas_study_count
  + 0.20 * OpenTargets genetic_association score
  + 0.10 * ancestry/sample-size support
  + 0.10 * novelty/recency of locus
```

For the two-day build, simplify to:

```text
genetic_support =
  max_normalized_neglog10_p
  + study_count_bonus
  + OpenTargets genetic_association score
```

### Functional Support

```text
functional_support =
  eQTL/colocalization score
  + cell_type_relevance
  + OpenTargets rna_expression / affected_pathway support
```

Cell-type relevance for Alzheimer:

```text
microglia: high
astrocytes: medium/high
oligodendrocytes: medium
neurons: medium/high depending on gene/pathway
endothelial/vascular: medium
```

### Literature Emergence

Inspired by the NIH predictive-breakthroughs paper.

```text
pct_new = papers published in current year or prior year / all topic papers
paper_growth = papers in last 2 years / papers in previous comparable period
locus_growth = new GWAS loci in last 2 years / all topic loci
influence = mean or sum relative_citation_ratio
topic_mixedness = entropy across pathway groups or evidence types
```

MVP formula:

```text
emergence_score =
  0.35 * pct_new
  + 0.25 * normalized_paper_growth
  + 0.20 * normalized_locus_growth
  + 0.10 * normalized_mean_rcr
  + 0.10 * topic_mixedness
```

Interpretation:

- high emergence = topic is active and changing
- high mixedness = topic connects several previously separate areas
- high RCR = influential papers exist in the topic

### Clinical Translation

```text
clinical_translation =
  max_phase_score
  + active_trial_count_score
  + enrollment_score
  + has_results_bonus
```

Phase mapping:

```text
NA/observational: 0.1
Early Phase 1: 0.2
Phase 1: 0.3
Phase 1/2: 0.4
Phase 2: 0.6
Phase 2/3: 0.75
Phase 3: 0.9
Phase 4/approved-like: 1.0
```

### Clinical Saturation

```text
clinical_saturation =
  total_trial_count
  + completed_or_failed_trial_count
  + total_enrollment
  + number_of_distinct_interventions
```

This helps separate:

- validated/crowded
- over-invested/fragile
- under-translated

### Opportunity Score

```text
opportunity_score =
  0.35 * genetic_support
  + 0.25 * functional_support
  + 0.25 * emergence_score
  - 0.15 * clinical_saturation
```

Do not hide the components. The UI should show why a topic is ranked highly.

## UI Specification

First screen should be the actual interactive map, not a landing page.

### Main Scatter/Map

Recommended axes:

```text
x-axis: genetic + functional support
y-axis: clinical translation
size: literature attention or paper count
color: pathway group
halo/ring: emergence score
opacity or border: confidence/completeness
```

Quadrants:

```text
High genetics, high trials:
validated / crowded translational space

High genetics, low trials:
opportunity zone

Low genetics, high trials:
hypothesis-risk or legacy investment zone

Low genetics, low trials:
background or early weak signal
```

### Topic Detail Drawer

When selecting a topic, show:

```text
topic label
topic trajectory over time
paper count by year
top papers
top GWAS studies
top variants/loci
top genes
cell-type/eQTL support
pathways
clinical trials
score breakdown
data provenance links
```

### Filters

```text
pathway group
evidence type
trial phase
active vs completed trials
new/emerging loci
GWAS-only / eQTL-supported / trial-linked
hide APOE/amyloid dominant topics
```

### Ranked Lists

Side panel:

```text
Top under-translated genetic topics
Top saturated trial topics
Top emerging topics
Top high-genetics/high-functional-support genes
Top clinical-trial-heavy but weak-genetics areas
```

## Two-Day Sprint Plan

### Day 1 Morning: Data Ingestion

Build scripts:

```text
ingest_gwas_catalog.py
ingest_pubmed.py
ingest_icite.py
ingest_opentargets.py
ingest_clinical_trials.py
```

Targets:

- Pull GWAS Catalog Alzheimer studies by EFO trait.
- Pull associations for each study.
- Extract seed PMIDs.
- Add manual seed papers.
- Fetch PubMed metadata, references, and cited-by lists for seeds.
- Fetch iCite metrics for all PMIDs.
- Pull Open Targets associated targets for `MONDO_0004975`.
- Pull ClinicalTrials.gov Alzheimer studies.

Day 1 morning done when:

```text
data/processed/papers.parquet exists
data/processed/gwas_studies.parquet exists
data/processed/gwas_associations.parquet exists
data/processed/genes.parquet exists
data/processed/trials.parquet exists
```

### Day 1 Afternoon: Network And Evidence Joins

Build scripts:

```text
build_paper_network.py
cluster_topics.py
link_topics_to_evidence.py
score_topics.py
```

Tasks:

- Build bibliographic coupling edges.
- Build current-state co-citation edges if enough cited-by data is available.
- Cluster papers.
- Create topic labels.
- Join topics to GWAS associations through PMIDs.
- Join genes to Open Targets scores.
- Join genes to pathway groups.
- Join pathways to trials using manual mechanism mapping.
- Compute scores.

Day 1 afternoon done when:

```text
data/processed/topic_years.parquet exists
data/processed/topic_members.parquet exists
data/processed/evidence_edges.parquet exists
data/app/map_data.json exists
```

### Day 2 Morning: Interactive UI

Build:

- Static Vite/React app, or single-page D3/Plotly app.
- Use local JSON exports.
- No backend required for MVP.

Minimum UI:

- scatter/map
- filters
- topic detail drawer
- ranked topic list
- evidence links

### Day 2 Afternoon: QA And Demo Narrative

QA:

- Click 5 topics and verify evidence makes sense.
- Check APOE/amyloid dominance does not collapse the map.
- Check GWAS/trial counts are not empty.
- Check score components are visible.
- Check each topic has provenance links.

Demo story:

1. Show the whole map.
2. Explain axes.
3. Select amyloid as saturated/high translation.
4. Select microglia/immune genetics as high genetics, emerging, lower translation.
5. Select a weak-genetics/high-trial or diagnostic topic as contrast.
6. Show trajectory over time.
7. Show evidence chain from papers -> GWAS loci -> genes -> pathway -> trials.

## Implementation Notes

### Cache Every API Response

Use deterministic file names:

```text
data/raw/gwas_catalog/studies_Alzheimer_disease_page_000.json
data/raw/gwas_catalog/associations_GCST007331.json
data/raw/pubmed/elink_refs_35379992.json
data/raw/pubmed/elink_citedin_35379992.json
data/raw/icite/icite_batch_000.json
data/raw/opentargets/disease_targets_MONDO_0004975.json
data/raw/clinical_trials/alzheimer_page_000.json
```

This prevents repeated API calls and makes debugging possible.

### Use Stable IDs

Canonical IDs:

```text
paper: PMID if available, else DOI, else internal hash
variant: rsID if available, else chr:pos:allele
gene: Ensembl ID if available, else HGNC symbol
disease: MONDO_0004975 for Alzheimer disease
trial: NCT ID
```

### Avoid Future Leakage In Claims

If using current co-citation, current RCR, or current citation counts, label outputs as current-state.

For historical/predictive claims, use only data available up to that year:

```text
paper.year <= snapshot_year
citing_paper.year <= snapshot_year
trial.start_date <= snapshot_year
GWAS publication_date <= snapshot_year
```

### Control Dominant Topics

APOE and amyloid can dominate the network.

Options:

- cap edge weights
- normalize by citation/reference count
- allow "hide APOE/amyloid" filter
- use pathway-specific submaps
- show saturated topics separately

### Manual Review Is Acceptable

For the two-day prototype, manually curated files are fine:

```text
manual_seed_papers.csv
gene_pathway.csv
intervention_mechanism.csv
topic_label_overrides.csv
```

The key is provenance. Mark manual mappings as manual.

## Definition Of Done

The prototype is successful if it can show:

1. A literature-derived topic map of dementia/AD genetics research.
2. Topic clusters with paper trajectories over time.
3. GWAS loci and genes attached to those topics.
4. Functional/eQTL support attached where available.
5. Clinical trial activity attached by pathway/mechanism.
6. A ranked list of under-translated, genetically supported, emerging topics.
7. Evidence provenance for every score shown in the UI.

## Initial Build Priority

If time is tight, prioritize in this order:

1. GWAS Catalog studies and associations.
2. PubMed seed expansion and paper metadata.
3. Bibliographic coupling network.
4. Topic clustering.
5. Open Targets associated target scores.
6. ClinicalTrials.gov trial table.
7. Manual gene pathway and intervention mechanism mapping.
8. Score calculation.
9. Interactive map.
10. Co-citation and yearly split/merge trajectories.

Co-citation is valuable, but the MVP can still work with bibliographic coupling plus yearly paper/GWAS/trial growth. Add full co-citation once the core evidence graph is working.

## Source Links

- GWAS Catalog API: https://www.ebi.ac.uk/gwas/docs/api
- GWAS Catalog REST root: https://www.ebi.ac.uk/gwas/rest/api
- Open Targets GraphQL API: https://platform-docs.opentargets.org/data-access/graphql-api
- Open Targets endpoint: https://api.platform.opentargets.org/api/v4/graphql
- ClinicalTrials.gov Data API: https://clinicaltrials.gov/data-api/api
- NCBI E-utilities: https://www.ncbi.nlm.nih.gov/books/NBK25501/
- NIH iCite API: https://icite.od.nih.gov/api
- Europe PMC REST API: https://europepmc.org/RestfulWebService
- Semantic Scholar Graph API: https://api.semanticscholar.org/api-docs/graph
- OpenAlex Developers: https://developers.openalex.org/
- Reactome Content Service: https://reactome.org/ContentService/
- NIH predictivebreakthroughs repo: https://github.com/NIHOPA/predictivebreakthroughs
- AD/ADRD GWAS consensus paper: https://www.nature.com/articles/s41588-026-02583-1
- SingleBrain eQTL paper: https://www.nature.com/articles/s41588-026-02541-x

