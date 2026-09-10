# Phishing URL Classifier 

Trains and evaluate ML learning classifiers that distinguish phising URLs from ;egitimate ones, using the PhiUSIIL Phishing URL Dataset - 2024

## Dataset 
[PhiUSIIL Phishing URL Dataset] (https://archive.ics.uci.edu/dataset/967/phiusiil+phishing+url+dataset)

The dataset spans 235,000+ URLs with over 100,000+ ones thougtht to be phishing, 54 features covering both URL structure
- Length
- Subdomains
- Special characters
- Webpage-Source Signals
    - Title match
    - forms
    - images
    - JS/CSS counts


## Setup

```bash
python -m venv venv
venv\Scripts\activate  # Or source venv/bin/activate on Mac/Linux
pip install -r requirements.txt
jupyter notebook
```

Run ` phishing_url_classifier.ipynb` from top through to the bottom. The dataset is automatically fetched inside the first cell so there is no need to worry about that

## Results and Discussion

### Model Development

I chose to make use of 4 candidate models to train on the PhiUSIIL Phishing URL Dataset...
1. Logistic Regression
2. Random Forest
3. XGBoost
4. LightGBM

A 5-fold stratified cross-validation comparison showed all 4 models were near the performance ceiling (accuracy/F1/ROC-AUC/PR-AUC all >= 0.9999)

With Random Forest and LightGBM being tied at a perfect 1.000000 across every metric, XGBoost was marginally behind with a (0.999984 accuracy) across the metrics, and Logistic Regression last (0.999894 accuracy).

You can see that all of the models were still near-perfect!

RandomomisedSearchCV tuning of the top 2 (Random Forrest and LightGBM) did **not** meaningfully change this either - both reached ROC-AUC 1.000000 - confirming the dataset itself is the dominant factor, not the model choice.

On the held-out test set, both tuned Random Forest and LightGBM achieved 0 false positives and 0 false negatives!

**Random Forest was selected as the final model**, since the 2 were statistically indistinguishable on performance and Random Forest's tuned config of 200 trees, no boosting or learning-rate machinery made it the simpler of the 2 to reason about and intepret



### Interrogating my near-perfect result

A 0/0 error rate is a slight cause for scrutiny rather than celebration, `URLSimilarityIndex` - a feature measuring similarity to a list of known, legitimate domains, correlates at 0.8 with the label and ranks top by feature importance. However, removing it entirely barely changed performance... only changing the ROC-AUC by 0.000002, and feature importance is spread across a dozen-plus features; (`NoOfExternalRef`, `LineOfCode`, `NoOfSelfRef`, `NoOfImage`,
`IsHTTPS`, and others) rather than concentrated in one leaking column. A SHAP summary plot confirmed each of these contributes sensible, directionally-consistent signal. For example a low `URLSimilarityIndex` and absense of HTTPS both push predictions towards **phishing**


This near-ceiling performance is consistent with already publicised results on PhiUSIIL and reflects the richness/redundancy of its engineered features, not a training artifact that is specific to this particular pipeline

### URL-only feature subset

Most of the top-ranked features: `LineOfCode`, `NoOfExternalRef`, `NoOfImage`, `NoOfJS`, `HasSocialNet` require fetching and parsing the live webpage, which is unabailable in real-time, URL-only classification scenario: a browser screening a link before it even loads.

Retraining Random Forest on only the 18 features computable from the raw URL string along (length, subdomain count, character ratios, `IsHTTPS` , etc) excluding corpus- features like the `URLSimilarityIndex` and `TLDLegitimateProb`that require reference stats, not just the URL, gave a measurable drop from the full-feature model on PhiUIIL's own test split, quantifying the real cost of a fetch-free deployment rather than leaving it as an unverified caveat.


### Out-of-dist generalisation test

The URL-only model was evaluated agaist 30 URLS freshly pulled from OpenPhis's live feed and 30 well-known legitimate domains (google.com, apple.com, gov.uk, etc).

Despite near-perfect in-dataset performance, the model classified **every** URL in this test, phishing and legitimate alike, as **Phishing**!!

ROC-AUC of 0.96 showed the underlying prob score still ranked legitimate URLS above phishing ones on average, but all scores were compressed near 0, (legitimate mean probability 0.0185, max 0.056) rather than merely sitting under the 0.5 decision threshold.


Comparing feature distributions between legitimate training examples and the OOD legitimate set showed `DomainLength` was the largest gap (training mean 19.2 vs OOD mean 10.9) 

PhiUSIIL's "legitimate" class appears to reflect longer, likely crawled/indexed pages rather than bare homepage URLS, so a plian domain like `apple.com` falls outside the structural pattern the model has learned to associate with legitimate.

A robustness check varying the definition of "special chars" in the URL-only feature extraction  produced ROC-AUC ranging from 0.60 - 0.96 depeding on the formula used. Classification accuracy at the default threshold remained 0.50 under both definitions, every URL was classified as phishing.

Showing the core generalisation failure is robust to this uncertanty, even though though the ranking itself isn't


### Conclusions 
- Random Forest, tuned, is the recommended model for the PhiUSIIL dataset itself, withe performance effectively indistinguishable from LightGBM
- Near-prefect in-dataset performance should not be read as evidence of a generally strong phishing classifier; it substantially refeflects redundant, highly-informative features specific to that dataset
- A URL-only variant, more realistic for real-time deployment, performs measurably worse than the full-feature model and **fails to generalise** to plain URLS outsied the dataset, confidently missclassifying well-known legit domains as Phishing
- This points to a missmatch between PhiUSIIL's legitimate-class URL structure (likely crawled/indexed pages, code isn't public so have to theorise) and the broader space of real legit URLS, rather than a flow in the approach itself

### Limitations and future work
- The dataset is a static snapshot; phishing tactics evolve, so a production system would
  need periodic retraining and drift monitoring.
- The OOD test used a small sample (30 URLs per class); a larger, more systematic
  real-world evaluation set (and ideally a legitimate sample matched to PhiUSIIL's own
  collection methodology) would give a more statistically robust generalization estimate.
- Reproducing PhiUSIIL's exact feature-extraction formulas was not possible without
  access to the original extraction code; results involving re-derived features (the
  URL-only subset, the OOD test) should be read as approximate rather than exact.
- A natural next step would be retraining on a legitimate-URL sample that includes bare
  homepages alongside crawled subpages, to test whether the generalization gap closes


## UPDATE - Generalistaion gap: tested and closed on 10/09/2026

This hypothesis was based of what was above in the future work, Added 600 real legit domains from the Majestic Million top-sites list to the URL-only training set as a bare-homepage "legitimate" examples, then I retrained the model. Evaluated on a fresh, held-out set of URLS not used in training, performance improved from a 50% accuracy (0% legitimate recall - every URL classsified as a phish) to **94%** accuracy ROC-AUC 0.996), with legitimate recall reaching 1/00.

This confirms the original generalisation failure was caused by a gap in the traning distribution. PhiUSIIL's legitimate class under-represents bare homepage URLs, rather than a fundamental limitaion of the URL-only feature set or model choice.

The trade-off is a modest reduction in phishing recall (0.89 vs. 1.00 previously), meaning roughly 11% of phishing URLs are now missed rather than caught. This could be because some phishing URLs are also short and simple, and the augmented model has become
less aggressive about flagging brevity alone as suspicious. A larger, more balanced augmentation set and a precision/recall trade-off analysis would be a natural next step to recover phishing recall without reintroducing the legitimate-URL blind spot.

A specific technique worth adding is typosquatting detection: checking whether a domain is a near-match to a well-known brand with characters substituted for lookalikes (e.g. `micr0soft.com` for `microsoft.com`, using `0` for `o`). This would involve computing edit distance between the candidate domain and a curated list of high-value target domains, ideally after normalizing common homoglyph substitutions (`0↔o`, `1↔l`, `rn↔m`) so disguised variants are still caught. 

Notably, this is close to what `URLSimilarityIndex` the single most important feature identified in this project's earlier interpretability analysis — was likely already capturing at a higher level; a hand-built typosquatting feature would make that signal explicit and auditable rather than an opaque learned
similarity score, and could specifically target the short, brand-adjacent phishing URLs the augmented model now under-detects.