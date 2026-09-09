# TrustRank

An implementation of the **TrustRank** algorithm (Gyöngyi, Garcia-Molina & Pedersen,
*"Combating Web Spam with TrustRank"*) adapted to **fraud analytics on payment
networks**. Instead of ranking web pages by how far they sit from trusted seed
pages, it ranks payment accounts by how much trust propagates to them from a
small, oracle-vetted set of seed accounts — so that accounts controlled by
fraudsters end up with low trust scores.

## Idea

1. Model payments as a directed graph: an edge `Sender → Receiver` exists if the
   sender ever paid the receiver.
2. Pick a small set of **seed** accounts that are well-connected in the *reverse*
   graph (money flows back through them), using **inverse PageRank**.
3. Ask an **oracle** whether each candidate seed is trustworthy. Here the oracle
   is a ground-truth list of known bad senders — any seed on that list is
   rejected.
4. Starting from the vetted good seeds, propagate trust through the transition
   matrix with a damping factor. Trust attenuates with distance, so accounts that
   are only reachable through long or fraud-adjacent paths accumulate little or
   no trust.
5. Sort accounts by trust score and inspect where the known bad senders land —
   they should cluster at the bottom (many at exactly `0`).

## Data

Two CSV files are expected (paths in the notebook point at Google Drive and
should be adjusted for local use):

| File | Columns | Purpose |
| --- | --- | --- |
| `Payments - Payments.csv` | `Sender`, `Receiver`, `Amount` | Transaction ledger used to build the graph |
| `bad_sender - bad_sender.csv` | `Bad Sender` | Ground-truth list of fraudulent accounts, used as the oracle and for evaluation |

Account identifiers from all three columns are label-encoded into a single
contiguous integer space, and a `reverse_mapping` is kept to translate encoded
node ids back to the original account numbers.

## Implementation

Everything lives in `TrustRank.ipynb`.

### Graph construction
- Payments are aggregated by `(Sender, Receiver)` (summing `Amount`).
- A `networkx.DiGraph` `G` is built with one edge per aggregated sender/receiver
  pair.

### Core functions

| Function | Description |
| --- | --- |
| `create_transition_matrix_from_networkx(graph)` | Column-stochastic transition matrix `M`; each edge `s → t` contributes `1 / out_degree(s)` at `M[t][s]`. |
| `create_inverse_transition_matrix_from_networkx(graph)` | Reverse-graph transition matrix; each edge `s → t` contributes `1 / in_degree(t)` at `M[s][t]`. |
| `inverse_pagerank(inverse_transition_matrix, alpha, num_iterations)` | Power iteration `score = α · Mᵀ_inv · score + (1 − α) · v₀` on the reverse graph. |
| `SelectSeed()` | Runs `inverse_pagerank` on `G` with `α = 0.85`, `20` iterations; returns per-node seed desirability. |
| `Rank(node_list, seed_desirability)` | Returns node indices sorted by descending desirability. |
| `O(node_index)` | Oracle: `0` if the node is in `bad_sender_list`, else `1`. |
| `TrustRank(transition_matrix, num_nodes, oracle_limit, alpha_bias, bias_iterations)` | Full algorithm (below). |

### `TrustRank` steps
1. `s = SelectSeed()` — seed desirability via inverse PageRank.
2. `sigma = Rank(...)` — candidate accounts ordered by desirability.
3. Take the top `oracle_limit` candidates; keep only those the oracle marks good,
   producing a static trust vector `d`.
4. Normalize `d` so the retained good seeds share equal, unit-sum weight.
5. Biased power iteration
   `trust = α_bias · (M · trust) + (1 − α_bias) · d`
   for up to `bias_iterations`, breaking early on convergence.

### Run configuration
```python
transition_matrix = create_transition_matrix_from_networkx(G)
trust_rank_values = TrustRank(transition_matrix, len(G.nodes),
                              oracle_limit=40, alpha_bias=0.85,
                              bias_iterations=100)
```

## Evaluation & output

- Accounts are sorted by trust score and translated back to original ids via
  `reverse_mapping`.
- The trust scores of every account in the ground-truth bad-sender list are
  printed. In the sample run all 20 known bad senders receive very low scores
  (roughly `0.007` down to `0.0`), with 8 of them at exactly `0` — i.e. no trust
  reaches them.
- Two histograms are produced with `matplotlib`:
  - distribution of trust scores across all accounts,
  - distribution of trust scores restricted to known bad senders,
  saved as `trust_score_distribution.png`.

## Requirements

```
numpy
pandas
networkx
scikit-learn
matplotlib
```

## Usage

1. Install the dependencies above (e.g. `pip install numpy pandas networkx scikit-learn matplotlib`).
2. Update the two `pd.read_csv(...)` paths at the top of `TrustRank.ipynb` to
   point at your local `Payments` and `bad_sender` CSVs.
3. Run all cells in `TrustRank.ipynb`.
4. Tune `oracle_limit`, `alpha_bias`, and `bias_iterations` in the run cell as
   needed for your graph.
