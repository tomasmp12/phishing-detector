# Phishing URL Detector

A machine learning model that takes a URL and predicts whether it is phishing, using **only the URL text** (it never visits the page).

Portfolio project in AI + cybersecurity. Current stage: baseline in a notebook, validated on an external dataset.

## How it works

```
URL → extrair_features() → 12 numeric features → XGBoost → phishing / legitimate
```

**Features extracted from the URL** (code names are in Portuguese):

| Feature | What it measures |
|---|---|
| `comprimento` | URL length in characters |
| `n_pontos` | Dots in the domain (many subdomains is suspicious) |
| `n_hifenes` | Hyphens in the domain (e.g. `paypal-secure-login.com`) |
| `n_digitos` | Digits in the URL |
| `tem_https` | Uses HTTPS |
| `tem_arroba` | Contains `@` (the browser ignores everything before it) |
| `tem_ip` | The domain is an IP address |
| `tem_login` | Contains the word "login" |
| `entropia` | Randomness of the domain (randomly generated domains have high entropy) |
| `n_palavras_suspeitas` | Words like "verify", "secure", "account", "update", "bank" |
| `tem_porta` | Has an explicit port (e.g. `:8080`) |
| `marca_falsa` | Mentions a known brand (PayPal, CTT, …) on a domain that isn't theirs |

Before extracting the features, the URL is normalized: the `www.` prefix is removed (see *Findings*).

## Data

- **Training:** [PhiUSIIL](https://archive.ics.uci.edu/dataset/967/phiusiil+phishing+url+dataset) (UCI, id 967). It has 235,795 URLs: 134,850 legitimate and 100,945 phishing. Stratified 80/20 split.
- **External test:** [PhishBench-External](https://data.mendeley.com/datasets/2ctjcnm9kn) (Mendeley). It has 1,000 URLs from July 2025: 500 phishing (PhishTank) and 500 legitimate (Tranco). The model never saw this data.

## Results (accuracy)

| Model | PhiUSIIL (test) | External |
|---|---|---|
| Original | 0.97 | **0.46** |
| Normalized URL (no `www.`) | 0.98 | **0.96** |
| Normalized, without the `tem_https` feature | 0.95 | 0.94 |

Final model (normalized), on the external test:

| | precision | recall |
|---|---|---|
| legitimate | 0.93 | 1.00 |
| phishing | 1.00 | 0.92 |

Confusion matrix: 499 legitimate correct and 1 false alarm; 461 phishing caught and 39 missed.

## Findings

**1. The model learned `www.`, not phishing.**
Accuracy on the training dataset was 97%. On the external dataset it dropped to **46%**, worse than chance: the model flagged almost every legitimate site as phishing.

The cause: in PhiUSIIL, **100%** of legitimate URLs start with `www.`, compared to 42% of phishing URLs. The model learned the rule "no `www.` → phishing". In the external dataset, only 0.2% of legitimate URLs have `www.`, so the rule breaks.

Removing `www.` before extracting features raised external accuracy from **0.46 to 0.96**.

**2. HTTPS is a strong signal, but the model doesn't rely on it alone.**
The `tem_https` feature accounts for ~93% of the importance. In both datasets, 100% of legitimate URLs use HTTPS, while phishing uses it 49% of the time (PhiUSIIL) and 85% (external). In PhiUSIIL, **every** URL without HTTPS is phishing, so that single question classifies half of the phishing URLs. Without this feature, the model still reaches 0.95 on PhiUSIIL and 0.94 on the external set, so the remaining features generalize well.

I kept the feature because "no HTTPS → suspicious" makes sense in the real world. Still, most current phishing already uses HTTPS, and the feature doesn't help in those cases.

**Lesson:** a high score on the training dataset proves nothing. You need to test on data from a different source.

## Limitations

- **External test is probably optimistic:** the Tranco legitimate URLs are homepages only (`https://dell.com`), short and without a path. Real legitimate URLs often have long paths (`dell.com/support/...`), and the model may confuse them with phishing.
- **Small external test:** only 1,000 URLs.
- **Only looks at the URL text:** it can't detect phishing hosted on compromised legitimate domains or on services like Weebly and Wix when the URL shows no obvious signs.
- **No temporal split:** PhiUSIIL train and test come from the same period. Phishing changes over time and the model ages.
- **No adversarial robustness:** it doesn't handle typosquatting (`paypa1.com`) or homoglyphs (`раypal.com` with Cyrillic letters).
- **Limited brand list:** the `marca_falsa` feature only knows 15 brands.

## Privacy

The model only needs the URL text and never visits the page. This makes it possible, in the future, to run it **locally in the browser** without sending browsing history to any server.

## How to run

```bash
git clone https://github.com/tomasmp12/phishing-detector.git
cd phishing-detector
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

1. The training dataset is downloaded automatically by the notebook (`ucimlrepo`).
2. For the external test, download the PhishBench-External CSV and save it as `data/phishbench_external.csv`.
3. Open `notebooks/01_baseline.ipynb` and run the cells **in order**.

The trained model is in `models/xgb_phishing.json`. To load it:

```python
from xgboost import XGBClassifier
model = XGBClassifier()
model.load_model("models/xgb_phishing.json")
```

## Project structure

```
phishing-detector/
├── notebooks/01_baseline.ipynb   # all the work: data, features, training, evaluation
├── models/xgb_phishing.json      # trained model
├── data/                         # datasets (not tracked by git)
└── requirements.txt
```

## Next steps

1. **Demo:** FastAPI API and web page.
2. **Production version:** temporal split, HTML content features, robustness against typosquatting and homoglyphs, MLOps (CI, Docker, drift monitoring) and a browser extension running the model locally with ONNX.
