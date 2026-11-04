# Optimal Trading Strategies in a Stochastic Discrete-Time Environment in Brazil

Code for the undergraduate thesis (TCC) of Murilo Fiorio Pires (Faculty of Economic Science, UFRGS), supervised by Prof. Nelson Seixas dos Santos.

Python implementation of the Samuelson (1969) consumption and portfolio choice model, calibrated with real Ibovespa and CDI data. The program computes the optimal portfolio (how much the investor should keep in stocks) and the fraction of wealth consumed in each period, and simulates how wealth evolves over time. Utility is CRRA, as in the thesis project.

The code is a pipeline in the *pipes and filters* style: each step receives the output of the previous one. Classes hold the state (the markets and the investor), and the model's equations are in functions that take numbers and return numbers. The full description is in [`docs/projeto/projeto.md`](docs/projeto/projeto.md) and the list of requirements in [`docs/requisitos/requisitos.md`](docs/requisitos/requisitos.md). The code, comments, notebooks and documents are in Portuguese.

## Installation

You need Python 3.12. Library versions are pinned in `requirements.txt`; install from it instead of installing packages one by one.

```bash
git clone https://github.com/ecompfin-ufrgs/optimal-trading-strategies-stochastic-discrete-time-environment-brazil.git
cd optimal-trading-strategies-stochastic-discrete-time-environment-brazil
```

```bash
pip install -r requirements.txt
```

## Running the model

```bash
python -m app --diario
```

The paper uses the daily database, `data/mercado_diario.db`, which is in the repository with real Ibovespa and CDI data.

The program also accepts monthly data, which is the default when you run it without `--diario`. The monthly database (`data/mercado.db`) is not in the repository, since the paper does not use it. Without it, the program runs on a synthetic series, only so the demonstration still works. To get real monthly data, build the database with the ingestion script (see "Updating the database").

The investor's parameters are passed on the command line, so you can try other values without editing the code. `python -m app --help` lists all options:

```bash
python -m app --anos 20 --beta-anual 0.90 --gamma 3.0
python -m app --diario --anos 10
python -m app --diario --graficos        # also writes figures to results/
```

Beta, the discount factor, is given per year, and the program converts it to the frequency of the data. With daily data it is the annual beta raised to 1/252, since a year has about 252 trading days. A beta of 0.96 per trading day, instead of per year, would describe an investor who consumes almost everything in the first year.

Beta does not change the optimal portfolio, only the consumption fractions, because it does not enter the first-order condition for alpha. The remaining horizon does not enter it either, so the decision is myopic: the share in stocks does not depend on how many periods are left.

Typical output of `python -m app --diario` (the program prints in Portuguese):

```
=== Esteira DP-CRRA-IID (Samuelson 1969) - base diaria ===
Ativos de risco       : ['ibov']
R_f (pregao)          : 0.00048924   (13.12% a.a.)
mu_hat (pregao)       : 0.00051104   (13.74% a.a.)
gamma                 : 5.0
Carteira otima a*     : [0.0584]
Phi_hat               : 0.998042
beta                  : 0.999838 por pregao  (0.96 a.a.)
theta_0 (pregao)      : 0.001024
theta_T (terminal)    : 1.0000
Consumo por ano (frac. de W_0), horizonte de 5 anos:
   ano 1: 0.2602
   ano 2: 0.2646
   ano 3: 0.2691
   ano 4: 0.2736
   ano 5: 0.2783
E[W_T] (T=1260)       : 0.001113  [P5=0.001074, P95=0.001155]
```

Consumption is reported as a sum per year. With daily data each consumption fraction is around 0.001 per trading day, and printing 1,260 numbers of that size would not say much. Summing within each year gives the fraction of W0 consumed in that year, which is the number usually reported.

This output uses the default of 4 million scenarios. The numbers and figures in the paper come from a run with 40 million, in which the optimal portfolio is 0.0430:

```bash
python -m app --diario --n-scenarios 40000000 --graficos
```

The reason for so many scenarios is Monte Carlo error. With 4 million, the optimal portfolio changes considerably from one seed to another (standard deviation of 0.0097 across 100 seeds); with 40 million, it falls to 0.0030. The comparison is in the notebook `tests/20_numeros_do_artigo.ipynb`.

## Result figures

```bash
python -m app --diario --graficos
```

This writes nine figures to `results/` in the same run as the model. The versioned ones are from the daily database, generated with the 40 million scenarios used in the paper. The figures take their parameters from the command line, so a plot cannot show one gamma while the text reports another.

| Figure | What it shows |
|---|---|
| `foc_G_de_alpha.png` | the curve G(alpha) crossing zero, where the optimal alpha is |
| `alpha_vs_gamma.png` | the optimal alpha falling as gamma rises (roughly in proportion to 1/gamma) |
| `alpha_vs_mu.png` | the optimal alpha rising with the expected return and crossing zero when it equals the CDI |
| `alpha_vs_sigma.png` | the optimal alpha falling as volatility rises |
| `theta_t.png` | the consumption fractions rising until they reach 1 (F13), on a log scale |
| `theta_vs_beta.png` | the fraction consumed in the first period falling as beta rises |
| `theta_vs_premio.png` | how the fraction consumed in the first period responds to the risk premium, one curve per gamma: it rises if gamma > 1, falls if gamma < 1 and does not change if gamma = 1 |
| `riqueza_W_t.png` | wealth over time, with the band between the 5th and 95th percentiles |
| `consumo_por_ano.png` | consumption summed per year |

The other two figures in the paper, `alpha_sementes.png` (the optimal portfolio for each seed) and `diagnostico_retornos.png` (daily returns, quantile-quantile plot and autocorrelations), come from the notebook `tests/20_numeros_do_artigo.ipynb`, which also writes them to `results/`.

Without the flag no file is written. This is on purpose: you can try `--gamma`, `--anos` and `--beta-anual` freely without overwriting the figures used in the paper.

The footer of each figure records the run that produced it: database, date window, R_f, gamma, beta (per year and per period), T, W0, number of scenarios and paths, seed and the resulting optimal alpha.

Five of these figures redo computations over the Monte Carlo scenarios, and [`projeto.md`](docs/projeto/projeto.md) explains which. With daily data and 4 million scenarios, a run with the flag takes about 1 minute, against a few seconds without it; with the 40 million used in the paper, about ten times longer.

## Updating the database

The database in the repository is enough to reproduce the paper. Daily returns in `data/mercado_diario.db` go from 2022-05-24 to 2026-08-05. The sources are Yahoo Finance for the Ibovespa and the Central Bank of Brazil SGS API for the CDI (series 12 for daily data and series 4391 for monthly data). The ingestion overwrites the database of the requested frequency.

To rebuild the versioned database with the same window:

```bash
python -m app.ingestao 2022-05-22 2026-08-06 --diario
```

The `retornos` table, which is the one the model reads, comes out identical to the versioned one. The only difference is in the `cdi` table, which gains 2026-08-06: the Central Bank API includes the requested end date and Yahoo does not, so that day has no Ibovespa return to match.

To download newer data, leave out the end date and the ingestion runs up to today:

```bash
python -m app.ingestao 2022-05-22 --diario
```

In that case the results are those of the new window, no longer the ones in the paper.

To build a monthly database (`data/mercado.db`), which the paper does not use, run the ingestion without `--diario`. With no dates, it starts on 2000-01-01:

```bash
python -m app.ingestao
```

## Structure

```
app/                      application package
  dal.py                  data access: download, SQLite, returns (F1, NF6)
  ingestao.py             builds the database from the original sources (F1, NF6)
  mercado.py              RendaFixa (CDI) and RendaVariavel (Ibovespa) (F2–F4)
  agente.py               CRRA investor (F5, F6, F8, F9)
  nucleo.py               the model's equations, in separate functions (F6–F11)
  principal.py            runs the steps in order (F10, NF5)
  graficos.py             result figures, outside the pipeline (F15)
  __main__.py             command line, python -m app (F14, NF2, NF5)
data/                     versioned SQLite database (daily)
results/                  figures written by --graficos (versioned)
docs/                     design and requirements documents
tests/                    20 notebooks, organized by requirement
requirements.txt          pinned library versions
LICENSE                   MIT license
```

## License

The code is under the MIT license. The full text is in [`LICENSE`](LICENSE).
