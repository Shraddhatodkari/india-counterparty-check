# India Counterparty Check

**A working due-diligence prototype that confirms who a company legally is, who owns it, whether it appears on a sanctions list, and what recent regulatory or legal news exists about it, with every fact linked to its source.**

**Live demo:** https://shraddhatodkari.github.io/india-counterparty-check/

> Independent project. Not a certified compliance product and not legal advice. The tool gathers and organises evidence; a person makes every decision.

---

## The problem

Before a bank, an investor or an advisor works with a company, someone has to answer four questions:

1. **Who is this company legally?** (registered name, registry number, address)
2. **Who owns or controls it?** (parent companies)
3. **Is it, or its parent, on a sanctions list?**
4. **Is there recent regulatory or legal trouble that a reviewer should know about?**

In many teams this is still done by hand: searching several government websites and news sites, copying results into a spreadsheet and writing a memo. It is slow, inconsistent between analysts and hard to evidence later.

## What the tool does

| Step | What happens | Technique | Source |
| --- | --- | --- | --- |
| 1. Resolve the legal entity | Finds the company's LEI, legal name, address and status. For Indian companies this includes the **CIN** from the Ministry of Corporate Affairs. | Registry lookup | GLEIF Global LEI Index |
| 2. Map ownership | Finds the direct and ultimate parent, or the reason none is reported, and draws the relationship graph. | Relationship records | GLEIF |
| 3. Screen sanctions lists | Screens the company **and its parents** against 24,406 listed names, allowing for spelling mistakes, word order and legal suffixes. | Fuzzy entity matching + a trained ML match model | OFAC SDN list, UN Security Council Consolidated List |
| 4. Compare identifiers | Checks whether the company's registry number or LEI appears in the matching list record. A shared identifier confirms the same legal entity, not just a similar name. | Identifier extraction from list remarks | OFAC SDN list |
| 5. Screen news (AI) | Finds recent regulatory and legal news and classifies each item by category, status and legal entity, with a link to the article. | Web search + AI language-model classification | Public news and official filings |
| 6. Apply policy rules | Applies seven written rules and records which were triggered. | Rules | Built-in policy |
| 7. Route to a person | Standard sign-off, reviewer, or senior reviewer. No model can approve or clear a company. | Rules | Built-in policy |

## Results on 11 real companies

All data was retrieved on **17 November 2026**.

| Outcome | Companies | Why |
| --- | --- | --- |
| **Escalate** | Sberbank | Registry number **1027700132195** appears in the OFAC SDN record: confirmed as the same legal entity (rule SAN-00) |
| **Review** | HDFC Bank, Tata Steel, Hindustan Unilever, Maruti Suzuki India, Nestle India | AI news screening found open regulatory or legal matters in the last 12 months (rule MED-01) |
| **Review** | Bosch | A parent exists but is not publicly disclosed (rule OWN-01) |
| **Standard** | Infosys, Reliance Industries, Larsen & Toubro, Siemens | Identity confirmed, no sanctions match, no open matter in the last 12 months |

Examples of what the analysis catches:

- **Cross-border parents:** Hindustan Unilever → Unilever PLC (UK), Maruti Suzuki India → Suzuki Motor Corporation (Japan), Siemens Limited → Siemens AG (Germany). Parents are screened too.
- **Subsidiary versus company:** Tata Steel's Dutch pollution prosecution is labelled as concerning a **subsidiary**, not Tata Steel Limited itself.
- **Different legal entity:** a Russian arbitration award against **Siemens Energy India Limited**, a separate demerged company, is set aside and **not attributed** to Siemens Limited.
- **Indian aliases:** OFAC lists "SBERBANK INDIA" and "SBERBANK MUMBAI" as aliases. Both are found.

## Name-matching evaluation

Real list names were changed on purpose to test whether the tool still finds them. Clean Indian company names from the GLEIF index were screened to count false alarms.

| Test | Result |
| --- | --- |
| One spelling mistake | 264 of 289 found (91.3%) |
| Words in a different order | 299 of 299 found (100%) |
| Different legal ending (e.g. "Pvt Ltd") | 300 of 300 found (100%) |
| 1,000 clean Indian company names | 1 flagged for review, 0 escalated |

## Machine-learning match model

A **logistic-regression** model (scikit-learn) estimates whether a candidate name pair is a true match. It uses seven explainable features: rule-based name score, share of exactly matching words, share of fuzzily matching words, number of rare exact words, word-count gap, average word rarity, and whether the names are identical letter-for-letter.

- Training data: 2,782 labelled name pairs. Positives are real list names with deliberate changes (spelling mistakes, word order, suffixes, dropped words, exact copies). Negatives are near-miss candidates, including candidates for 1,000 clean Indian company names.
- Test data: 1,154 name pairs from **separate** source names (grouped split, so no name appears in both sets).

| Held-out test | Rule only (score ≥ 80) | Match model |
| --- | --- | --- |
| Precision | 0.918 | **0.927** |
| Recall | 0.963 | **0.956** |
| F1 score | 0.940 | **0.941** |
| False alarms | 45 | **39** |
| Missed matches | 19 | **23** |
| ROC AUC | not applicable | 0.980 |

**Honest reading:** the model gives 13% fewer false alarms but misses 4 more true matches, so overall quality is about the same. For that reason the policy escalates on the rule score and on identifiers, and the model is shown as a second opinion. The model score is the likelihood that **two names** refer to the same listed entity. It is **not** the probability that a company is sanctioned.

## AI news screening

- **Retrieval:** one web search per company, combining its name with risk terms (penalty, probe, lawsuit and sector terms).
- **Classification:** an AI language model read the headlines and snippets and assigned each item a **category** (for example Securities regulation, Banking regulation, Tax dispute, Environmental), a **status** (for example "Penalty imposed, under appeal", "Allegation or proceeding open", "Resolved in company's favour") and the **legal entity** concerned (company, subsidiary, or a different legal entity).
- **Grounding:** every item links to its article, and items older than 12 months are marked.
- **Guardrails:** news can only send a case to a reviewer (rule MED-01). It can never escalate or clear a company.
- **Limitation:** classification is based on headlines and short snippets, not full articles, and has not been checked by a person.

## Policy rules

| Rule | Triggered when | Result |
| --- | --- | --- |
| SAN-00 | The company's registry number or LEI appears in a matching list record | Escalate: same legal entity confirmed |
| SAN-01 | Name score of 90 or more, without a shared identifier | Escalate to a senior reviewer |
| SAN-02 | Name score of 80 to 89 | Reviewer must clear it |
| ID-01 | The LEI registration is not current | Reviewer checks identity |
| ID-02 | The entity status is not active | Reviewer checks identity |
| OWN-01 | A parent exists but is not publicly disclosed | Ask the company for its ownership chart |
| MED-01 | AI news screening finds an open matter from the last 12 months | Reviewer reads the linked articles |

## Technology

| Area | Technology |
| --- | --- |
| Entity resolution | Text normalisation, IDF word weighting, Damerau-Levenshtein fuzzy matching, identifier comparison |
| Machine learning | scikit-learn logistic regression; weights exported and run in the browser |
| AI | Language-model classification of retrieved news, with source links |
| Data pipeline | Python (OFAC, UN and GLEIF downloads and parsing); Node.js (screening run and evaluation) |
| Front end | HTML, CSS and JavaScript, no framework; SVG relationship graph |
| Hosting | GitHub Pages (static; nothing typed into the page leaves the browser) |

## Known limitations

- Screens the **OFAC and UN** lists only. EU, UK and Indian lists are not yet included.
- Ownership comes from **GLEIF accounting parents**. It does not show individual beneficial owners.
- **Court, insolvency (NCLT) and GST records** are not connected.
- **Companies only.** Directors and owners are not screened.
- News screening covers the 11 sample companies only, uses headlines and snippets, and is not human-verified.
- The match model was trained on deliberately changed list names, not on real analyst decisions.
- All data is a **snapshot** from 17 November 2026.

## Roadmap

- Add the EU and UK consolidated sanctions lists
- Screen directors and key people
- Add Indian sources: MCA company master data, NCLT insolvency cases and GST registration status
- Run news screening live for any company, with full-article reading and human review of each classification
- Retrain the match model on real reviewer decisions

## Repository contents

| Path | Contents |
| --- | --- |
| `index.html` | The app: page, matching logic, trained model weights, policy rules and the 11 saved reports |
| `lists.json` | 10,386 sanctioned entities (10,112 OFAC, 274 UN) with 24,411 names and 6,125 records carrying identifiers |
| `source/` | The pipeline that produced the data, the model and the reports (see below) |

### Rebuild everything from the official sources

```bash
cd source
python build/fetch.py          # download OFAC and UN files into raw/
python build/lists.py          # parse them into site/lists.json
python build/gleif.py          # registry and ownership records for the 11 companies
node build/eval.js             # matching evaluation
node build/features.js         # labelled name pairs for the model
python build/train.py          # train and test the match model (scikit-learn)
node build/run.js              # run the checks and write site/reports.json
```

Then insert `site/matcher.js` and `site/reports.json` into `build/page.html` (placeholders `/*MATCHER*/` and `/*REPORTS*/`) to produce `index.html`. The news classifications are stored in `news/classified.json`.

### Run the page locally

```bash
python -m http.server 8000
# open http://localhost:8000
```

## Data sources

- [OFAC Specially Designated Nationals (SDN) List](https://sanctionssearch.ofac.treas.gov/), U.S. Department of the Treasury
- [UN Security Council Consolidated List](https://main.un.org/securitycouncil/en/content/un-sc-consolidated-list), United Nations
- [GLEIF Global LEI Index](https://search.gleif.org/), which carries each Indian company's CIN from the Ministry of Corporate Affairs
- News articles and official filings linked item by item in each report

## Author

**Shraddha Todkari** · AI for finance, due diligence and risk · Pune, India
