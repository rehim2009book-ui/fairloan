# TwinCheck: catching hidden age bias in loan decisions

**Track:** AI + Finance

**Team:** Proxy Hunters (Ughur Huseynli)

**Code:** [fairloan.ipynb](fairloan.ipynb) · [open it in Colab](https://colab.research.google.com/github/rehim2009book-ui/fairloan/blob/main/fairloan.ipynb) (Runtime → Run all, about two minutes, no API key needed)

**Demo video:** https://youtu.be/FBHCnEF82qk

**One-page write-up (PDF):** [TwinCheck_writeup.pdf](TwinCheck_writeup.pdf)

*This project is a prototype and is not financial advice.*

## The problem

Banks use computer models to decide who gets a loan, and the person who is refused rarely learns why. A model can also be unfair without anyone noticing: in the Netherlands, a tax-office algorithm that treated nationality as a risk sign wrongly accused tens of thousands of families of fraud, and the government resigned over it in 2021. The usual fix is "don't give the model the sensitive detail". We tested whether that works for age, for a young person applying for their first loan.

## What we built

**In:** one loan application (savings, job, housing, loan size and so on).
**Out:** the decision, the model's real reasons in plain words, the smallest change that would get a yes, and a check of whether a detail that mostly signals age changed the answer.

**Try it:** open the notebook in Colab and press Runtime → Run all. At the bottom a small form opens. Change one detail of an applicant, press Submit, and watch the answer change.

A real example from our data, applicant #470, who is under 25:

> **DECISION: REJECTED** (approve chance 40%)
> **Why:** "Your checking account balance was overdrawn below 0 DM, which hurt your application. Living rent-free where you pay no rent hurt your chances. Having been in your current job for less than 1 year also hurt you."
> **Twin check:** an identical applicant who rents instead of living rent-free gets a **YES**. The model was never told her age.
> **What would get a yes:** bring the checking account from below 0 to 0–200 DM, or build savings to 500–1000 DM.

## What we added

We started from suggested project **4.2.3, Fairness audit of a loan model**. The brief asks for a loan model and the approval-rate gap between two age groups. We asked a harder question: is the bias still there after age is deleted, and where is it hiding? Our own additions:

1. **Twin Test.** Two applicants identical except for one detail; we count how often the decision flips. The data picks the pairs.
2. **Age guesser.** A second model that tries to guess "under 25" without seeing age.
3. **Fair test.** Among people who really paid back, who is wrongly rejected more often?
4. **Two fixes judged on money**, with the dataset's own cost rule, not on accuracy.
5. **"What would get a yes"** for every rejected person. It never says "be older".
6. **AI explanations with a lie check:** do the notes match the model's real reasons?
7. **Our own user:** the refused applicant, not the bank.
8. **Corrected labels.** The dataset's official code table is wrong for 8 columns. We found this after our first explanations told an applicant the opposite of the truth, and switched to the corrected table (Grömping, 2019).

## How it works

**Data.** Statlog German Credit (UCI): 1,000 loan applicants, 20 details each, 700 good risks and 300 bad. 149 are under 25. We dropped two columns whose coding cannot be trusted (sex, foreign worker). We held back 20% as a test set; because that leaves only 28 under-25s, the fairness numbers use all 1,000 people with 5-fold cross-validation, where each person is scored by a model that never saw them.

**Model.** Logistic regression (scikit-learn). We also tried a random forest and kept the logistic regression because it cost the bank less and its reasons can be read directly from its weights. We score with the dataset's cost rule: approving a bad risk costs 5, rejecting a good one costs 1.

**Explanations.** We take each person's three strongest reasons from the model and ask Gemini to say them in plain words. Gemini does not make the decision. Its answers are saved in `explanations.json`, so the notebook runs without a key.

## Results

Our baseline, approving everyone, scored 70.0% accuracy and a cost of 300 on held-out test data. Our model scored 67.0% accuracy and a cost of **142**. By accuracy it looks worse than the baseline; by money it is twice as good, because it catches 68.3% of the bad risks.

| What we measured (all 1,000 people unless noted) | Result |
|---|---|
| Approval gap between 25+ and under-25s | 17.4 points with age, **13.8 with age deleted** |
| Age guesser, without seeing age | 0.78 (0.50 = coin flip, 1.00 = perfect); top clue: living rent-free |
| Twin Test, housing only (rent-free vs renting) | decision flips for 13% of applicants |
| Good payers wrongly rejected (age deleted) | **37.5% of under-25s vs 26.8% of 25+** |
| Fix A: also delete the details that leak age | cost 617 → 603; young good payers wrongly rejected 37.5% → 36.4% |
| Fix B: lower pass mark for under-25s | cost 617 → 708; young good payers wrongly rejected 37.5% → 27.3% |
| Rejected people with a small change that leads to a yes | about 9 in 10, in both age groups |
| Gemini *guessing* the model's reasons (30 people) | names 42% of them (random = 18%) |
| Gemini *told* the model's reasons (30 people) | every note gave the three real reasons and added none |

**Main takeaway:** deleting the age column does not remove age bias, because other details give age away. The bias we found is modest, it can be measured, and fixing it has a price.

**Where it fails, and why we think so.**

- **There is no free fix.** Fix A costs nothing but barely helps. Fix B makes wrongful rejections equal, but costs the bank about 15% more, uses age on purpose, and under-25s then get approved slightly more often than older people.
- **Small groups.** Only 88 young good payers. Our age results are strong signs, not proof.
- **A rule we cannot explain.** The model learned that owning real estate *hurts* an application. The label is correct; the rule is just strange. Our explainer reports it honestly, which would confuse an applicant.
- **Our own lie check was wrong at first.** Its keyword matching flagged 9 of 30 correct notes (it read "unknown amount of savings" as the loan amount). We read them by hand and fixed the patterns.
- **Old data.** 1970s Germany, only loans that were granted, bad risks over-sampled. Money details still decide most of each answer, which is how it should be.

## What we'd do next

1. Add reliability ranges to every gap by re-sampling the data, because the young group is small.
2. Run the same audit on a modern lending dataset, and package TwinCheck as one function that audits any model.

## Sources

- **Data:** Hofmann, H. (1994). Statlog (German Credit Data). UCI Machine Learning Repository. https://doi.org/10.24432/C5NC77 (CC BY 4.0)
- **Corrected code table:** Grömping, U. (2019). South German Credit Data: Correcting a Widely Used Data Set. http://www1.beuth-hochschule.de/FB_II/reports/Report-2019-004.pdf
- **Ideas we built on:** paired testing for discrimination in software, Galhotra, Brun and Meliou (2017), https://arxiv.org/pdf/1709.03221 ; checking AI explanations against a model, https://arxiv.org/html/2412.10220v1
- **Background:** Amnesty International (2021), "Xenophobic Machines", https://www.amnesty.org/en/latest/news/2021/10/xenophobic-machines-dutch-child-benefit-scandal/ ; EU AI Act, Annex III (credit scoring is high-risk), https://artificialintelligenceact.eu/annex/3/
- **Libraries:** pandas, NumPy, scikit-learn, matplotlib, ucimlrepo; google-genai and Gradio (optional).
- **AI tools:** Gemini API (`gemini-3.5-flash-lite`) writes the plain-language explanations inside the project. We used Claude to help plan the project, research the background, draft and debug the code, and run the notebook. We read and tested every part ourselves.
