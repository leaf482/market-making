# Market Making Lab

Market Making Lab is a self-paced course in market making, financial mathematics, and modern C++. The work is an educational simulation. It is a place to practice quantitative development by typing every implementation yourself.

The course starts in Python so the mathematics and the accounting stay visible. After the strategies and tests are validated, the same models are ported to C++.

## Current state

Documentation only. No strategy, simulator, or test source has been created. Every milestone in [LEARNING_LOG.md](LEARNING_LOG.md) is `not_started`.

Implementation begins at milestone M1.1 after you approve the start of Phase 1. The tutor stops after each step until you confirm that you typed the code, ran the command, and checked the result.

## How the course works

The working agreement is [.cursor/rules/tutor.mdc](.cursor/rules/tutor.mdc). In short:

1. The tutor explains the goal, the reason, and the mathematics or programming idea.
2. The tutor names the file and shows the small section to type.
3. You type the code and run the command yourself.
4. You compare the result with the success criterion in [PROJECT_PLAN.md](PROJECT_PLAN.md).
5. The next step starts only after your confirmation.
6. A milestone is recorded in [LEARNING_LOG.md](LEARNING_LOG.md) only after you confirm it.

Explanations are in Korean. Source code, identifiers, comments, and these documents are in English.

## Roadmap

| Phase | Focus | Outcome |
| --- | --- | --- |
| 1 | Quotes, execution, inventory, cash, and PnL | Hand-checked ledger |
| 2 | Fixed-spread market maker | Python quotes, fills, and accounting |
| 3 | Inventory-based quotes | Skew, position limits, and a shared price path |
| 4 | Stochastic simulation | Seeded Brownian motion and Poisson arrivals |
| 5 | Avellaneda–Stoikov | Reservation price, optimal spread, and one simulated path |
| 6 | Strategy evaluation | Fixed, inventory-based, and Avellaneda–Stoikov on shared randomness |
| 7 | C++ | Port, tests against the Python fixtures, and a timing measurement |

The completion criteria, formulas, and numerical fixtures are in [PROJECT_PLAN.md](PROJECT_PLAN.md).

## Documents

- [PROJECT_PLAN.md](PROJECT_PLAN.md) — milestones, formulas, fixtures, and reading.
- [LEARNING_LOG.md](LEARNING_LOG.md) — confirmation record. It stays empty of completed work until you confirm a milestone.
- [.cursor/rules/tutor.mdc](.cursor/rules/tutor.mdc) — tutoring protocol.

## Later source layout

You create these files yourself when a milestone names them. They are not part of the initial repository.

```text
python/
  quotes.py
  accounting.py
  strategies.py
  simulation.py
  backtest.py
  test_quotes.py
  test_accounting.py
  test_strategies.py
  test_simulation.py
  test_backtest.py
  requirements.txt
cpp/
  CMakeLists.txt
  src/
  tests/
```

Phases 2 and 3 use the Python standard library. NumPy enters in Phase 4 for random number generation. Phase 7 uses C++20 and the C++ standard library.

From the repository root, after the Python tests exist, you run:

```text
python -m unittest discover -s python -v
```

## Reading

The implementation target is the closed-form approximation in:

Avellaneda, M., and Stoikov, S. (2008). High-frequency trading in a limit order book. *Quantitative Finance*, 8(3), 217–224. <https://doi.org/10.1080/14697680701381228>

The rest of the reading list is in [PROJECT_PLAN.md](PROJECT_PLAN.md).

## Scope

The lab covers market microstructure, inventory risk, stochastic simulation, statistical evaluation, and C++ implementation of those models.

The lab keeps a single-process simulator, plain modules, deterministic tests, and a small C++ port. Live trading, exchange connectivity, user interfaces, databases, and cloud infrastructure are outside the course.
