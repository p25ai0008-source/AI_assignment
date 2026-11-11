Classical AI Reasoning Agent
============================



1. Files
--------
reasoning_agent.ipynb   Complete implementation: data loading and exploration, tune / report splits, parser,
                        reasoning engines, router, evaluation, experiments, reasoning traces, test files,
                        submission, final summary (accuracy and inference time).
report.pdf              Representation, algorithms, data discipline, experiments, reasoning traces, limitations.
submission.csv          Predictions for the private test. Columns: id,label  (label is exactly True or False).
README.txt              This file.

Files written by the notebook when it runs:
private_test.csv / private_test_labels.csv     private test (built from ProverQA if no private test is present)
public_test.csv  / public_test_labels.csv      logic-only test built from the held-out public data
submission.csv                                 predictions for the private test
submission_public.csv                          predictions for the public logic test


2. How to run (Google Colab or any Jupyter, Python 3.10+)
---------------------------------------------------------
1. Upload reasoning_agent.ipynb and, if available, the test file. In Section 1 set
       TEST_PATH = "private_test.csv"
   If an older private_test.csv from a previous run is present, delete it first.
2. Runtime -> Run all.
     Section 12 answers the test files and prints the accuracy of each one (when labels are available).
     Section 13 prints the combined accuracy on the public data and the inference time of every test file.
3. If TEST_PATH does not exist and USE_PROVERQA_AS_PRIVATE_TEST = True, the private test is built from ProverQA
   (dev split, 100 problems per level: easy, medium, hard; labels verified by the Prover9 prover); its answers
   are written to private_test_labels.csv.
4. RUN_EXPERIMENTS = False (default) skips the experiments of Section 10, which take several minutes.
   An error inside one experiment is reported and the notebook continues.
5. The notebook never stops on missing data. If a public dataset cannot be downloaded (no internet, no token),
   only its evaluation is skipped; the test file is still answered and submission.csv is still written.
6. FOLIO on Hugging Face is gated. To evaluate on it in Colab, accept its terms on the dataset page and store a
   READ token in Colab "Secrets" under the name HF_TOKEN (the token is never written in the code). Without a
   token the notebook falls back to the public FOLIO file on GitHub. The token is not needed to answer a test file.


3. Input format
---------------
File type : CSV, JSON, JSONL or parquet.  One row = one problem.

Columns (detected automatically, case-insensitive):
    id             : id, example_id, question_id, qid, idx              optional (row number if missing)
    knowledge base : context, premises-FOL, premises_fol, premises, kb, knowledge_base, theory,
                     facts_rules, given_info, facts                     required
    query          : statement, conclusion-FOL, conclusion_fol, conclusion, query, question,
                     hypothesis, goal                                   required
    label          : label, gold, answer, flag, target                  optional (used only to print accuracy)

Knowledge base cell, any of:
    a JSON list string        ["Kind(bob)", "∀x (Kind(x) → Nice(x))"]
    a Python list string      ['Kind(bob)', '∀x (Kind(x) → Nice(x))']
    one string                formulas separated by newlines, ';' or '. '   (not by commas)
    a real list               (JSON / JSONL / parquet)

Query  : one formula.
Labels : True / False (also 1 / 0, yes / no, entailment). Unknown or Uncertain counts as False.
Output : submission.csv with columns id,label; label is True or False.

Every input must be formal logic. The notebook checks this and prints
"Input format check: all inputs are formal logic" or the number of inputs that do not look like logic.


4. Sample test file (CSV)
-------------------------
id,context,statement
q1,"[""Young(bob)"", ""Kind(bob)"", ""∀x (Young(x) → Smart(x))"", ""∀x (Smart(x) ∧ Kind(x) → Successful(x))""]",Successful(bob)
q2,"[""Quiet(anne)"", ""Big(bob)"", ""∀x (Big(x) ∧ ¬Quiet(x) → Strong(x))""]",¬Strong(anne)
q3,"[""∀x (Kind(x) ∧ Young(x) → ¬Smart(x))"", ""Kind(bob)"", ""Young(bob)""]",Smart(bob)
f1,"[""∀x (Drinks(x) → Dependent(x))"", ""∀x (Drinks(x) ⊕ Jokes(x))"", ""∀x (Jokes(x) → ¬Unaware(x))"", ""Unaware(rina)""]",Dependent(rina)
f2,"[""∀x (Dog(x) → Brown(x) ∨ Black(x))"", ""Dog(rex)""]",Black(rex)
f3,"[""∃x (Cat(x) ∧ Black(x))"", ""∀x (Cat(x) → Pet(x))""]",∃x (Pet(x) ∧ Black(x))
p1,"[""Rain → Wet"", ""Sprinkler → Wet"", ""Rain ∨ Sprinkler""]",Wet
p2,"[""Rain → Wet""]",Wet → Rain
c1,"[""Blaf := ¬Ziklo"", ""Vork := Blaf ∧ Trune"", ""Lumbo := Vork"", ""Ziklo"", ""Trune""]",do(¬Ziklo) → Lumbo
c2,"[""Gonk := Hupt ∨ Dask"", ""Crov := Gonk ∧ ¬Dask"", ""Hupt"", ""Dask""]","do(¬Hupt, ¬Dask) → Gonk"

Expected answers: q1 True, q2 True, q3 False, f1 True, f2 False, f3 True, p1 True, p2 False, c1 True, c2 False.

Sample row in JSONL (single-string knowledge base, no id):
{"premises": "Tall(a); a = b", "hypothesis": "Tall(b)", "label": "True"}

Sample submission.csv:
id,label
q1,True
q2,True
q3,False


5. Accepted notations
---------------------
Unicode FOL      ∀x (Young(x) → Smart(x))
ASCII / words    forall x (Young(x) -> Smart(x))      young(x) implies smart(x)
Prover9          all x (Young(x) -> Smart(x)).
Prolog / ProbLog smart(X) :- young(X).                0.9::young(bob).
TPTP             fof(r1, axiom, ![X]: (young(X) => smart(X))).
LaTeX            \forall x (Young(x) \rightarrow Smart(x))
Textbook         (∀x)(Young(x) ⊃ Smart(x))
RuleTaker triple ("Bob" "is" "young" "+")
FLD              (x): {A}x -> {B}x
z3py             ForAll(x, Implies(Young(x), Smart(x)))
Logic-LM / Pyke  Young($x, True) >>> Smart($x, True)
SMT-LIB          (assert (forall ((x U)) (=> (Young x) (Smart x))))
Lean             ∀ x, Young x → Smart x
Set notation     Young ⊆ Smart        bob ∈ Young
Causal models    V := e,  V = e,  V <- e,  e -> V,  "X causes V",  priors V ~ Bern(p);
                 queries do(...) → Y,  P(Y | do(...)),  Y_{x},  X □→ Y
Different rows of one file may use different notations.


6. Method in one paragraph
--------------------------
A dialect normalizer and a hand-written recursive-descent parser turn every notation into one internal AST of
immutable tuples. A router chooses the engine from the structure of the knowledge base, without any dataset flag:
causal syntax goes to a structural causal model (abduction, intervention, prediction); rule bases (literal facts
and rules with at most one variable per rule) whose vocabulary contains every query symbol are answered under the
closed-world assumption by backward chaining with negation as failure (forward chaining builds the proof trees);
a Horn KB with a positive literal query goes to forward chaining; every other KB is answered under the open-world
assumption by resolution refutation (CNF, Skolemisation, Tseitin-style naming, unification, set of support,
best-first clause selection, deterministic budget of 50 given clauses). Every failure is caught: an unreadable
formula is dropped, an unreadable query returns False, and the reason is recorded.


7. Data discipline
------------------
No model is trained. The public datasets are used to build and test the parser, to choose a few settings, and to
measure accuracy. Every dataset is split into a tune part (exploration and choice of settings) and a report part
(accuracy only), and the notebook checks that they share no context, story or problem:
    RuleTaker     tune: dev split          report: test split
    FOLIO v2      tune: train              report: validation
    CounterBench  tune: 600 (by problem)   report: 600 held out (all 5 renamed copies of a problem on one side)


8. Results (deterministic; times depend on the machine)
-------------------------------------------------------
Report splits
    RuleTaker test (18,000 sampled, 6 depth configs)      100.00 %
    FOLIO v2 validation (203)                              85.22 %
    CounterBench held-out (600)                            93.33 %
    Combined (18,803)                                      99.63 %
Tune splits (for reference)
    FOLIO v2 train (1,001)                                 81.12 %
    CounterBench tune (600)                                94.17 %
Logic-only test files
    Private test, ProverQA (300)                           99.33 %   (easy 100.00, medium 98.00, hard 100.00)
    Public logic test (2,003)                              96.51 %   (RuleTaker 1,200 + FOLIO 203 + CounterBench 600)
    Built-in test (26)                                    100.00 %
Outside checks (public data, not used for tuning)
    ProverQA dev (1,500)                                   99.40 %
    FLD.v2 test (5,000)                                    96.58 %
    ProofWriter OWA test (600)                             84.50 %   (closed-world convention of RuleTaker kept)
Generated test (264 problems, 13 notations, labels from z3 / truth tables / brute force)    99.62 %
Random propositional formulas checked against a truth table                                  346 / 346 identical
The CounterBench errors reported as engine failures are copies of one cyclic causal graph; the notebook prints
the cycle instead of silently answering False.


9. Assumptions
--------------
* Output is binary: Unknown / Uncertain counts as False (not entailed).
* The test file has no dataset or semantics flag: closed vs. open world is decided from the structure of the KB.
* Constants and predicate names are case-insensitive (Bob = bob).
* Free single-letter variables in the KB are universally quantified; in Prolog / TPTP input every capitalised
  argument is a variable. In a query, a free variable is existential if the query is a combination of literals,
  universal if the query is rule-like (contains → or ↔).
* Quantifier scope: "Qx (...)" covers only the bracket unless x is used after it; otherwise it extends right.
* A contradictory KB does not make every query true (set of support does not use the contradiction).


10. Dependencies
----------------
Reasoning : Python standard library only (re, itertools, heapq, difflib, functools).
Data      : pandas, datasets, huggingface_hub (pre-installed in Colab).
