# Detetor de phishing a partir do URL

Modelo de machine learning que recebe um URL e diz se é phishing, usando **apenas o texto do URL** (não visita a página).

Projeto de portefólio em IA + cibersegurança. Fase atual: baseline em notebook, com teste num dataset externo.

## Como funciona

```
URL → extrair_features() → 12 pistas numéricas → XGBoost → phishing / legítimo
```

**Pistas extraídas do URL:**

| Pista | O que mede |
|---|---|
| `comprimento` | Número de caracteres do URL |
| `n_pontos` | Pontos no domínio (muitos subdomínios é suspeito) |
| `n_hifenes` | Hífenes no domínio (ex.: `paypal-secure-login.com`) |
| `n_digitos` | Dígitos no URL |
| `tem_https` | Usa HTTPS |
| `tem_arroba` | Tem `@` (tudo o que vem antes é ignorado pelo browser) |
| `tem_ip` | O domínio é um endereço IP |
| `tem_login` | Contém a palavra "login" |
| `entropia` | Aleatoriedade do domínio (domínios gerados ao acaso têm entropia alta) |
| `n_palavras_suspeitas` | Palavras como "verify", "secure", "account", "update", "bank" |
| `tem_porta` | Indica uma porta explícita (ex.: `:8080`) |
| `marca_falsa` | Menciona uma marca conhecida (PayPal, CTT, …) num domínio que não é o dela |

Antes de extrair as pistas, o URL é normalizado: o `www.` é removido (ver *O que descobri*).

## Dados

- **Treino:** [PhiUSIIL](https://archive.ics.uci.edu/dataset/967/phiusiil+phishing+url+dataset) (UCI, id 967). São 235.795 URLs, dos quais 134.850 legítimos e 100.945 phishing. Divisão 80/20 estratificada.
- **Teste externo:** [PhishBench-External](https://data.mendeley.com/datasets/2ctjcnm9kn) (Mendeley). São 1.000 URLs de julho de 2025: 500 phishing (PhishTank) e 500 legítimos (Tranco). O modelo nunca viu estes dados.

## Resultados (accuracy)

| Modelo | PhiUSIIL (teste) | Externo |
|---|---|---|
| Original | 0,97 | **0,46** |
| Com URL normalizado (sem `www.`) | 0,98 | **0,96** |
| Normalizado e sem a pista `tem_https` | 0,95 | 0,94 |

Modelo final (normalizado), no teste externo:

| | precision | recall |
|---|---|---|
| legítimo | 0,93 | 1,00 |
| phishing | 1,00 | 0,92 |

Matriz de confusão: 499 legítimos certos e 1 falso alarme; 461 phishing apanhados e 39 falhados.

## O que descobri

**1. O modelo aprendeu o `www.` e não o phishing.**
No dataset de treino, a accuracy era de 97%. No dataset externo caiu para **46%**, pior do que o acaso: o modelo marcava quase todos os sites legítimos como phishing.

A causa: no PhiUSIIL, **100%** dos URLs legítimos começam por `www.`, contra 42% dos de phishing. O modelo aprendeu a regra "sem `www.` → phishing". No dataset externo, só 0,2% dos legítimos têm `www.`, e a regra falha.

Removendo o `www.` antes de extrair as pistas, a accuracy no externo subiu de **0,46 para 0,96**.

**2. O HTTPS é um sinal forte, mas o modelo não depende só dele.**
A pista `tem_https` tem ~93% da importância. Nos dois datasets, 100% dos legítimos usam HTTPS, e o phishing usa-o em 49% (PhiUSIIL) e 85% (externo). No PhiUSIIL, **todos** os URLs sem HTTPS são phishing, por isso basta essa pergunta para classificar metade do phishing. Sem esta pista, o modelo continua com 0,94 no externo, por isso as restantes pistas fazem quase todo o trabalho.

Mantive a pista porque a regra "sem HTTPS → suspeito" faz sentido no mundo real. Mesmo assim, a maioria do phishing atual já usa HTTPS, e para esses casos ela não ajuda.

**Lição:** um resultado alto no próprio dataset não prova nada. É preciso testar em dados de outra fonte.

## Limitações

- **Teste externo provavelmente otimista:** os legítimos do Tranco são só páginas iniciais (`https://dell.com`), curtas e sem caminho. URLs legítimos reais costumam ter caminhos longos (`dell.com/support/...`), e o modelo pode confundi-los com phishing.
- **Teste externo pequeno:** são só 1.000 URLs.
- **Só olha para o texto do URL:** não deteta phishing alojado em domínios legítimos comprometidos ou em serviços como Weebly e Wix sem sinais óbvios no URL.
- **Sem split temporal:** treino e teste do PhiUSIIL são da mesma época. O phishing muda depois e o modelo envelhece.
- **Sem robustez adversarial:** não trata typosquatting (`paypa1.com`) nem homóglifos (`раypal.com` com letras cirílicas).
- **Lista de marcas limitada:** a pista `marca_falsa` só conhece 15 marcas.

## Privacidade

O modelo só precisa do texto do URL e não visita a página. Isto permite, no futuro, corrê-lo **localmente no browser**, sem enviar o histórico de navegação para nenhum servidor.

## Como correr

```bash
git clone https://github.com/tomasmp12/phishing-detector.git
cd phishing-detector
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

1. O dataset de treino é descarregado automaticamente pelo notebook (`ucimlrepo`).
2. Para o teste externo, descarrega o CSV do PhishBench-External e guarda-o como `data/phishbench_external.csv`.
3. Abre `notebooks/01_baseline.ipynb` e corre as células **por ordem**.

O modelo treinado está em `models/xgb_phishing.json`. Para o carregar:

```python
from xgboost import XGBClassifier
model = XGBClassifier()
model.load_model("models/xgb_phishing.json")
```

## Estrutura

```
phishing-detector/
├── notebooks/01_baseline.ipynb   # todo o trabalho: dados, pistas, treino, avaliação
├── models/xgb_phishing.json      # modelo treinado
├── data/                         # datasets (não incluídos no git)
└── requirements.txt
```

## Próximos passos

1. **Demo:** API em FastAPI e página web.
2. **Versão a sério:** split temporal, pistas do conteúdo HTML, robustez contra typosquatting e homóglifos, MLOps (CI, Docker, monitorização de drift) e uma extensão de browser com o modelo local em ONNX.