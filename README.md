# claim_topics_dataset

This repository contains the claims topic dataset extracted from ClaimsKG. You will find the procedure to extract the topic sample from ClaimsKG, transform the samples into the individual annotator files and to reconcile divergences between annotators to produce the final dataset. 



We provide the original annotations of each annotator and the reconciled gold standard dataset. 



# Extraction from ClaimsKG

## Normalization of keywords with thesauri

The entity linking of claims, headlines, reviews with DBPedia entities is performed in ClaimsKG (with TagMe) during the extraction step from fact-checking sites. Although this provides some degree of normalization, the coverage is fairly low and the entities aren't of a consistent granularity, which is why an additional normalization step is performed through the annotation with high-level social sciences thesauri (TheSoz and the UNESCO thesaurus). This allows to overlay a concept hierarchy on-top of the keywords in order to distinguish between higher and lower-level concepts and to normalize similar keywords into single entities. 

The annotation is performed with a dictionary matching approach, insensitive to minor surface morphological variation and word order for compounds that is  very similar to that of MGREP used in the NCBO Bioportal Annotator. The implementation of this reconciliation procedure is integrated directly in the upstream ClaimsKG generator program and is enabled with the 

We provide a turtle version of this normalized ClaimsKG, which can be used to reproduce the dataset. The easiest way to load the dataset and to run queries against it, is to use the Virtuoso docker image and to place the turtle file in the `toLoad`directory of mounted base data directory, as described here: <https://hub.docker.com/r/tenforce/virtuoso/>.



## Target Concepts

On the basis of [topic_counts.html](topic_counts.html), we selected the top-level concepts from the Thesauri that were the most frequent. When there were overlapping concepts, the concept from the TheSoz thesaurus was preferred. 

The concepts retained are the following:

- Helathcare: <http://lod.gesis.org/thesoz/concept_10045504>
- Taxes: <http://lod.gesis.org/thesoz/concept_10038824>
- Education: <http://lod.gesis.org/thesoz/concept_10035091>
- Immigration: <http://lod.gesis.org/thesoz/concept_10041774>
- Elections: 	<http://lod.gesis.org/thesoz/concept_10034501>
- Crime: <http://vocabularies.unesco.org/thesaurus/concept407>
- Environment: <http://lod.gesis.org/thesoz/concept_10058252>

For each concept, we can extract the all corresponding claims with all associated meta-information by using a SPARQL query. The query filters some non-thematic keywords with explicit regular expressions, retains only english-language claims and excludes AfrikaCheck claims, where the keywords are very noisy. You can see the query below



```
PREFIX skos:<http://www.w3.org/2004/02/skos/core#>
PREFIX thesoz: <http://lod.gesis.org/thesoz/>
PREFIX unesco: <http://vocabularies.unesco.org/thesaurus/>
PREFIX schema: <http://schema.org/>
PREFIX dct: <http://purl.org/dc/terms/>

SELECT ?claim ?review_url str(?claim_author_name) as ?claimer ?text str(?headline) as ?headline ?keywords  ?claim_date WHERE 
{
  {
    SELECT ?claim (group_concat(?kwlabel, ',') as ?keywords)  WHERE {
      ?claim schema:keywords ?keyword.
      ?keyword schema:name ?kwlabel.
       FILTER (!regex(str(?kwlabel),"immigration|education|economy|taxes|health care|Public Health|ASP Article","i"))

   } GROUP BY ?claim} 

  ?claim schema:keywords ?keyword.
  ?keyword dct:about ?kwc.
  ?keyword schema:name ?kwlabel.
  ?kwc skos:prefLabel ?kwcl_r.

  ?claim schema:text ?text_r.

  ?claim schema:author ?claim_author.
  ?claim_author schema:name ?claim_author_name.
  ?claim schema:datePublished ?claim_date.

  ?cr schema:itemReviewed ?claim.
  ?cr schema:author ?author.
  ?cr schema:headline ?headline.
  ?cr schema:url ?review_url

  BIND(str(?text_r) as ?text)
  FILTER (lang(?kwcl_r) = 'en')
  FILTER (regex(str(?kwc), "URI_OF_THE_CONCEPT","i"))
FILTER(!regex(str(?author), "http://data.gesis.org/claimskg/organization/africacheck"))
}

```

We ran this query through the virtuoso SPARQL interface to get the results as a TSV file for each concept. The set of all seven TSV files was used as the basis for the sampling and the generation of the files for annotation. You may find said files in the [extracted_claims](./extracted_claims) directory. 





## Annotation Protocol 

The annotation files for each annotator are provided in [individual_annotations](./individual_annotations). Each CSV annotation file was annotated in a local spreadsheet program, by putting any symbol in the column for each relevant concept. 

The annotators were asked to use only the information present in the file as much as possible (claim, headline, author, date) and to use a search engine if they were unfamiliar with particular entities or acronyms. The keywords pertaining to the target topics were removed, but the other keywords were left in place as they could provide useful information. 

For each concept, detailed guidelines were provided, with positive and negative examples. You may find a few examples below. 



### elections

This tag should be assigned if a claim deals with an ongoing or past election or the election system. It should not be assigned if the claim was only uttered in the context of an election, even if its implicit meaning is related to an election, e.g. when a candidate makes a statement about their opponent to belittle them or a candidate presents their campaign pledge. 

#### Positive examples:

- [https://www.politifact.com/factchecks/2012/jun/20/debbie-wasserman-schultz/billionaire-koch-brothers-gave-8-million-wisconsin/
  ](https://www.politifact.com/factchecks/2012/jun/20/debbie-wasserman-schultz/billionaire-koch-brothers-gave-8-million-wisconsin/)Billionaire Koch brothers gave $8 million to Wisconsin Gov. Scott Walker recall campaign, Dem chair says
- <http://www.politifact.com/north-carolina/statements/2019/feb/15/chuck-mcgrady/nc-republican-half-right-about-gop-support-redistr/> (l374)
  NC Republican half-right about GOP support for redistricting reform 
- <https://www.politifact.com/factchecks/2011/feb/27/leo-berman/state-rep-leo-berman-says-hawaii-governor-cant-fin/> 
  State Rep. Leo Berman says Hawaii governor can't find anything that says Obama was born in Hawaii 
  (the claim is about Leo Berman’s proposed bill to require presidential candidates to show their birth certificates and the connected doubts about Obama’s birthplace)

#### Negative examples:

- <https://www.politifact.com/factchecks/2012/aug/20/connie-mack/connie-mack-bill-nelson-voted-taxes-150-times/> 
  Connie Mack said Bill Nelson voted to raise taxes 150 times
- <https://www.politifact.com/factchecks/2010/sep/03/barack-obama/obama-says-tom-barrett-has-held-line-property-taxe/> (l36)
  Milwaukee Mayor Tom Barrett 'has held the line on property taxes.'
- <https://www.politifact.com/factchecks/2012/oct/17/mitt-romney/romney-says-obama-also-has-investments-chinese-com/> (l7)
  Romney says Obama also has investments in Chinese companies and through a Cayman Islands trust
- <http://www.politifact.com/florida/statements/2010/jul/26/rick-scott/rick-scott-touts-7-7-7-plan-create-700000-jobs-sev/> (l125)
  'My 7-step plan' creates 700,000 jobs in 7 years.
- <http://www.politifact.com/truth-o-meter/statements/2019/jul/02/donald-trump/cory-booker-and-drug-maker-campaign-cash-numbers/> (l445)
  Says Cory Booker has accepted over $400,000 from the pharmaceutical industry during his political career.

### taxes

This tag should be assigned if a claim is about taxes directly. If a concept is mentioned that can be related to taxes (i.e. government spending) but the connection to taxes is background knowledge rather than connected to the message of the claim, this tag should not be assigned. 

#### Positive examples:

- <https://www.politifact.com/factchecks/2015/jan/05/chain-email/what-will-happen-taxes-january-2015-chain-email-wr/> (l22)
  New tax increases that went into effect on Jan. 1, 2015, 'all passed under the Affordable Care Act, aka Obamacare.'
- <http://www.politifact.com/truth-o-meter/statements/2012/oct/05/mitt-romney/mitt-romney-says-barack-obama-provided-90-billion/> (l169)
  Mitt Romney says Barack Obama provided $90 billion in green energy 'breaks' in one year
  (this refers to tax breaks)
- <http://www.politifact.com/truth-o-meter/statements/2010/aug/10/john-boehner/rep-john-boehner-proposes-us-stop-stimulus-spendin/> (l402)
  'There's still about $400 billion or $500 billion of the stimulus plan that has not been spent. Why don't we stop it.'
  (This refers to a specific stimulus program consisting of tax cuts, thus the speaker proposes to stop these tax cuts)

#### Negative examples:

- <http://www.politifact.com/georgia/statements/2011/sep/27/paul-broun/broun-stimulus-money-funded-effort-will-kill-jobs/> (l152)
  Stimulus money funded a government board that made recommendations that would cost 378,000 jobs and $28.3 billion in sales.
  (maybe this stimulus money is connected to tax cuts, we don’t know and it is not essential to the claim because the speaker criticises the target of the funding which is unrelated to taxes)

### healthcare

This tag should be assigned to claims dealing with the healthcare system or health issues in general. 

#### Positive examples:

- <http://www.politifact.com/texas/statements/2016/apr/22/jerry-jones/pants-fire-jerry-jones-says-absurd-firmly-link-foo/> (l426) 
  Says it’s 'absurd' to say there’s enough data to establish a link between playing football and Chronic Traumatic Encephalopathy.
- <http://www.politifact.com/wisconsin/statements/2012/nov/25/jon-erpenbach/erpenbach-says-walker-misled-federal-health-care-l/> (l429) 
  Says Gov. Scott Walker has 'led people to believe that if Wisconsin doesn’t implement a (health-care) exchange, Obamacare doesn’t happen here.'
- <http://www.politifact.com/truth-o-meter/statements/2013/jun/26/barack-obama/barack-obama-says-when-he-went-college-los-angeles/> (l641)
  As a student at Occidental College in Los Angeles from 1979 to 1981, 'there were days where folks couldn't go outside. … because of all the pollution in the air.'

#### Negative examples:

- <http://www.politifact.com/truth-o-meter/statements/2019/jul/02/donald-trump/cory-booker-and-drug-maker-campaign-cash-numbers/> (l445)
  Says Cory Booker has accepted over $400,000 from the pharmaceutical industry during his political career.
  (this is not about healthcare but rather about industry)
- <http://www.politifact.com/texas/statements/2011/mar/07/gail-collins/new-york-times-columnist-gail-collins-say-texas-ra/> (l544)
  New York Times columnist Gail Collins say Texas ranks third in teenage pregnancies and first in repeat teen pregnancies
  (pregnancy is connected to healthcare but this claim is about education, not health-related issues)
- <http://www.politifact.com/truth-o-meter/statements/2014/aug/07/shelley-moore-capito/will-epa-regulations-stop-plants-burning-coal-shel/> (l208)
  'What (the Obama administration is) going to come out with in the next several months is you're not even going to be able to burn coal very limitedly in the existing plants.'
  (environmental issues are always almost also health issues; however, this claim does not address the health perspective)

 ## Agreement and reconciliation



Krippendorff’s α (Masi distance) All annotators:  0.75

__Pairwise agreement:__

A1 A2 0.84

A1 A3 0.66

A1 A4 0.81

A1 A5 0.78

A2 A3 0.67

A2 A4 0.85

A2 A5 0.81

A3 A4 0.67

A3 A5 0.65

A4 A5 0.78



The reconciliation script will be made available in the final version.

The final reconciled dataset can be found in `gold_updated.csv`.