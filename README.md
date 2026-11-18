# Phishing URL Detector

A machine learning model that takes a URL and predicts whether it is phishing, using **only the URL's domain** (it never visits the page).

Portfolio project in AI + cybersecurity. Current stage: baseline in a notebook, validated on an external dataset.

## How it works

```
URL → normalize (drop www. and path) → 15 numeric features → XGBoost → phishing / legitimate
```

**Features** (code names are in Portuguese):

| Feature | What it measures |
|---|---|
| `comprimento` | Length of `scheme://domain` |
| `n_pontos` | Dots in the domain (many subdomains is suspicious) |
| `n_hifenes` | Hyphens in the domain (e.g. `paypal-secure-login.com`) |
| `n_digitos` | Digits in the domain |
| `tem_https` | Uses HTTPS |
| `tem_arroba` | Contains `@` (the browser ignores everything before it) |
| `tem_ip` | The domain is an IP address |
| `tem_login` | Contains the word "login" |
| `entropia` | Randomness of the domain (randomly generated domains have high entropy) |
| `n_palavras_suspeitas` | Words like "verify", "secure", "account", "update", "bank" |
| `tem_porta` | Has an explicit port (e.g. `:8080`) |
| `marca_falsa` | Mentions a known brand (PayPal, CTT, …) on a domain that isn't theirs |
| `tld_suspeito` | Top-level domain often abused for phishing (`.top`, `.xyz`, `.lat`, …) |
| `alojamento_gratuito` | Hosted on a free site builder or hosting service (`weebly.com`, `wixsite.com`, `github.io`, …) |
| `encurtador` | URL shortener (`bit.ly`, `tinyurl.com`, …) |

## Data

- **Training:** [PhiUSIIL](https://archive.ics.uci.edu/dataset/967/phiusiil+phishing+url+dataset) (UCI, id 967). It has 235,795 URLs: 134,850 legitimate and 100,945 phishing. Stratified 80/20 split.
- **External test:** [PhishBench-External](https://data.mendeley.com/datasets/2ctjcnm9kn) (Mendeley). It has 1,000 URLs from July 2025: 500 phishing (PhishTank) and 500 legitimate (Tranco). The model never saw this data.

## Results (accuracy)

| Version | PhiUSIIL (test) | External |
|---|---|---|
| Original (full URL) | 0.97 | **0.46** |
| Without `www.` | 0.98 | 0.96 ⚠️ inflated, see finding 2 |
| Domain only (without `www.` and path) | 0.93 | 0.73 |
| + `tld_suspeito` | — | 0.81 |
| **+ `alojamento_gratuito` and `encurtador` (final)** | **0.94** | **0.83** |
| Final, without `tem_https` | 0.91 | 0.83 |

Final model, on the external test:

| | precision | recall |
|---|---|---|
| legitimate | 0.76 | 0.96 |
| phishing | 0.94 | 0.70 |

Confusion matrix: 478 legitimate correct and 22 false alarms; 352 phishing caught and 148 missed.

## Findings

The biggest lesson of this project: **the model kept learning quirks of the dataset instead of phishing.** It found three shortcuts, and the first two only showed up when I tested on data from a different source.

**1. It learned `www.`.**
On PhiUSIIL, accuracy was 97%. On the external dataset it dropped to **46%**, worse than chance: almost every legitimate site was flagged as phishing.

The cause: in PhiUSIIL, **100%** of legitimate URLs start with `www.`, compared to 42% of phishing URLs. The model learned "no `www.` → phishing". In the external set, only 0.2% of legitimate URLs have `www.`. Removing `www.` before extracting features fixed this.

**2. It learned "has a path → phishing".**
After fixing `www.`, the external accuracy was 0.96, but the model rated `github.com/tomasmp12/phishing-detector` as 100% phishing. In PhiUSIIL, **0%** of legitimate URLs have a path (they are all bare homepages), compared to 27% of phishing URLs. The external test had the same bias, because its legitimate URLs are also homepages, so the 0.96 was inflated.

Fix: the model now only looks at `scheme://domain`. The honest external accuracy dropped to 0.73. Adding three domain-based features (`tld_suspeito`, `alojamento_gratuito`, `encurtador`) raised it to **0.83**.

**3. HTTPS dominates feature importance, but the model doesn't need it.**
In both datasets, 100% of legitimate URLs use HTTPS, so every URL without HTTPS is phishing. That's why `tem_https` takes ~86% of the importance. Still, a model trained without it gets the same 0.83 on the external set, so the other features carry the real signal. I kept it because "no HTTPS → suspicious" is a reasonable real-world rule, even though most phishing today already uses HTTPS.

**Lesson:** a high score on the training dataset proves nothing. Test on data from a different source, and try URLs by hand.

## Limitations

- **Data snooping:** the three newer features were inspired by looking at the external test's errors, so the 0.83 is somewhat optimistic. The feature lists come from general knowledge of abused TLDs and hosting services, not only from the test URLs, but a clean measurement needs a third, unseen dataset.
- **Small external test:** only 1,000 URLs.
- **Domain only:** it can't detect phishing on compromised legitimate domains (most of the remaining misses) or signals in the path (`/login.php`, `/verify`).
- **No temporal split:** PhiUSIIL train and test come from the same period. Phishing changes over time and the model ages.
- **No adversarial robustness:** it doesn't handle typosquatting (`paypa1.com`) or homoglyphs (`раypal.com` with Cyrillic letters).
- **Hand-made lists:** brands, TLDs and hosting services are short manual lists that need maintenance.

## Privacy

The model only needs the URL's domain and never visits the page. This makes it possible, in the future, to run it **locally in the browser** without sending browsing history to any server.

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

1. **Clean evaluation:** a third test set (fresh OpenPhish phishing and legitimate URLs with paths).
2. **Demo:** FastAPI API and web page.
3. **Production version:** temporal split, HTML content features, robustness against typosquatting and homoglyphs, MLOps (CI, Docker, drift monitoring) and a browser extension running the model locally with ONNX.