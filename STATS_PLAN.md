# Method Match: frontend-only statistics learning app plan

## Purpose

Build a small web app that helps introductory statistics students choose an analysis for a research question. The first release is a matching exercise: learners read a short study scenario and match it to the best introductory statistical method, then see *why* that method fits.

The central lesson is that a label such as “survey” or “experiment” does not determine the analysis on its own. The choice depends on the question, outcome and predictor types, number of groups, whether observations are paired, and whether the researcher wants a comparison, association, or model. Study design still matters for interpretation: choosing a test cannot turn an observational association into a causal effect. [Penn State STAT 500 overview of variables and their roles](https://online.stat.psu.edu/stat500/Lesson01)

**Audience:** undergraduate students taking a first or second statistics course, including students preparing to analyze their own research projects. The app is a practice aid, not a tool that analyzes uploaded data or approves a real study's analysis plan.

## What learners should be able to do

After one round, a learner should be able to:

1. Identify the outcome, predictor or grouping variables, and the question being asked.
2. Distinguish independent groups from repeated measurements on the same people or objects.
3. Recognize when the outcome is continuous, binary, or absent because the task is qualitative.
4. Explain a plausible introductory method choice in one sentence.
5. Name one condition to check before using that method on real data.

## MVP boundaries

### Included

- One responsive page with a short introduction, a practice deck, and a round summary.
- Ten authored scenario cards with methods that can be reused across cards; there is no one-to-one elimination trick.
- Eight answer choices: independent-samples t-test, paired t-test, one-way ANOVA, Pearson correlation, simple linear regression, chi-square test of independence, binary logistic regression, and **No listed method fits**.
- A single best introductory answer per card, plus immediate explanatory feedback. Clearly defensible alternatives can be acknowledged without marking the learner simply wrong.
- A **Try again** action, a **Next scenario** action, and a **New round** action. No timer, lives, streaks, or account.
- Mouse, touch, and keyboard interaction. A learner can select a method by clicking a button; drag-and-drop can be added only if it remains an optional way to perform the same action.
- All content and scoring in the browser. No backend, Netlify Functions, login, uploaded data, or external AI call.

### Deferred

- Data entry, calculations, charts of real results, and assumption diagnostics.
- Adaptive tutoring, AI-generated scenarios, instructor accounts, classroom reporting, and a content management system.
- Broad method coverage such as nonparametric tests, mixed-effects models, survival analysis, and Bayesian methods.
- Detailed coverage of sampling, power, multiple comparisons, and causal inference. Feedback can flag these issues without pretending the MVP teaches them fully.

## Learner experience

1. **Start:** Show “Read the question, inspect the variables, choose a method.” A two-sentence example explains that the same survey can lead to different methods.
2. **Inspect a scenario:** Present the study description and a compact “What we know” panel: research aim, outcome, predictor or groups, and independent versus paired observations. The panel exposes facts already present in the story; it does not reveal the answer.
3. **Make a match:** The learner selects one reusable method button. A visible **Check match** button lets them change their selection before submitting.
4. **Learn from feedback:** Show **Best fit**, **Also defensible**, or **Reconsider**. Explain the decisive clues, a likely confusion, and one condition to check with actual data. Keep the original scenario and selected answer visible.
5. **Continue and review:** At the end, show the number of scenarios completed, a list of methods to revisit, and links to replay any card. A small “method map” summarizes the key decision cues.

The deck order may be shuffled, but method choices remain in a stable order for accessibility. Multiple cards use the same method. Learners never have to remember whether a label was “used up.”

## Initial method map

This is the learner-facing scope, not a universal decision tree. All methods require context and checks before use. [Penn State STAT 500 technique review](https://online.stat.psu.edu/stat500/Lesson12)

| Method | Introductory question it answers | Distinguishing cue and useful check |
| --- | --- | --- |
| Independent-samples t-test | Do two independent groups differ in mean continuous outcome? | Different people or objects in each group; check independence and whether the outcome's distribution supports the chosen t procedure. |
| Paired t-test | Is the mean *within-pair difference* different from zero? | Two measurements on the same unit or matched pairs; inspect the distribution of differences. |
| One-way ANOVA | Do mean continuous outcomes differ among three or more independent groups? | One categorical grouping variable with at least three levels; check independence and residual behavior. |
| Pearson correlation | What is the direction and strength of a linear association between two quantitative variables? | Descriptive, symmetric association question; inspect a scatterplot for nonlinearity and outliers. |
| Simple linear regression | How does the expected continuous outcome change with one predictor? | A named response and predictor, with a slope as the target; inspect the relationship and model residuals. |
| Chi-square test of independence | Are two categorical variables associated in a contingency table? | Counts in categories, no adjusted model requested; check independent observations and expected cell counts. [Penn State chi-square overview](https://online.stat.psu.edu/statprogram/reviews/statistical-concepts/chi-square-tests) |
| Binary logistic regression | How do predictor values relate to the odds of a yes/no outcome? | Binary response, potentially several predictors or adjustment; examine model fit and whether observations are independent. [Penn State binary logistic regression overview](https://online.stat.psu.edu/stat504/Lesson06) |
| No listed method fits | Is the question outside this introductory palette? | An open-ended thematic question, for example, needs a qualitative analysis plan; do not force a numeric test. |

For two independent groups, a regression with a group indicator can answer a closely related mean-comparison question. One-way ANOVA and regression with category indicators are likewise related. With two quantitative variables, correlation and regression summarize different aspects of the same relationship. The exercise should identify the **simplest method matching the stated aim**, while feedback names reasonable alternatives instead of implying the alternatives are invalid.

## Seed scenario deck

These are fictional teaching scenarios. Each card's final copy must state its target clearly and include enough design information to judge the match. Assume the ordinary method conditions suggested by the story are plausible; the feedback still reminds learners to check them with real data.

| ID | Scenario and precise question | Best introductory match | Decisive clue |
| --- | --- | --- | --- |
| 01 | Students are randomly assigned to one of two tutoring formats. Each student takes one numeric final exam. Do the formats differ in **mean score**? | Independent-samples t-test | Two independent groups; continuous outcome. |
| 02 | The same students take a numeric skills test before and after a workshop. Is the **mean within-student change** different from zero? | Paired t-test | Each student supplies a before/after pair. |
| 03 | Students are randomly assigned to one of three revision formats. Each takes one numeric exam. Is there evidence that **any group mean differs**? | One-way ANOVA | Three independent groups; continuous outcome. |
| 04 | A survey records sleep hours and concentration score for each respondent. How **strong and in what direction is their linear association**? | Pearson correlation | Two quantitative measures; association strength is the target. |
| 05 | An observational study records weekly revision hours and final score for each student. Estimate the **mean score change per extra revision hour**. | Simple linear regression | A continuous outcome, one quantitative predictor, and a slope. |
| 06 | A large survey records year of study (first, second, third) and preferred study location (home, library, café). Are the **two categorical variables associated**? Each respondent is counted once; expected cell counts are adequate. | Chi-square test of independence | A three-by-three table of independent counts. |
| 07 | A cohort records whether each student completes an online course within 30 days (yes/no), weekly study hours, and prior grade average. Estimate the **association of study hours with completion odds after adjusting for grade average**. | Binary logistic regression | Binary outcome plus adjusted predictors. |
| 08 | Different participants are randomly assigned to work with music or in silence. Each has one numeric task-completion time. Do the groups differ in **mean time**? | Independent-samples t-test | A second example showing that the method is reusable. |
| 09 | Independent seedlings receive varying measured amounts of light. At a fixed day, each has one height measurement. Estimate the **mean height change per additional hour of light**, within a roughly linear range. | Simple linear regression | Continuous response and a slope for one predictor. |
| 10 | Researchers conduct open-ended interviews about why students leave a course. Their aim is to **identify themes in the accounts**, with no numeric outcome or category counts planned. | No listed method fits | This is a qualitative thematic question as stated. |

Card 02 is intentionally a method choice, not evidence that the workshop caused improvement: practice, time, and other changes could also matter. Card 05 similarly asks for an observational slope, not a causal effect. Card 06 avoids a binary response and adjustment request, making chi-square a clearer introductory choice than logistic regression.

## Feedback and answer rules

- The primary answer gets **Best fit** and a two- or three-sentence explanation built from the case details.
- An authored `alsoDefensible` choice gets **Also defensible** and an explanation of what question or model framing would make it useful. Treat it as a successful match for completion; show the primary choice too.
- Other choices get **Reconsider** with a case-specific hint, then allow a retry. After a second attempt, reveal the best fit and explanation so a learner cannot get trapped.
- The summary reports *cards understood on the first attempt* and cards to revisit. It does not rank students or assign a grade.
- Every scenario includes a “Before using this with real data” note. For example, a paired t-test card should point to the distribution of paired differences; a chi-square card should point to expected counts.

Example for card 02: choosing the independent-samples t-test yields, “The outcome is numeric, but the two scores come from the same students. Match each student's after score to their before score and analyze the differences.” Choosing the paired t-test yields, “Best fit: the question is about the mean of those within-student differences. Check the differences before applying the test.”

The content file should explicitly store accepted alternatives and feedback rather than infer them from keywords. Before release, an introductory statistics instructor should review each card for ambiguity, assumptions, and plain-language accuracy.

## Content and frontend design

Use a Vite, React, and TypeScript single-page app. Keep the authored method definitions and scenarios in local typed data so reviewers can edit them in the repository. A card can follow this shape:

```ts
type Scenario = {
  id: string;
  title: string;
  prompt: string;
  clues: {
    aim: string;
    outcome: string;
    predictorOrGroups: string;
    observations: string;
  };
  bestMethodId: string;
  alsoDefensible: Array<{ methodId: string; explanation: string }>;
  explanation: string;
  conditionToCheck: string;
  reconsider: Record<string, string>;
};
```

Use stable method IDs rather than matching on displayed labels. On startup, validate that every referenced method exists, every card has feedback for its likely distractors, and no card's best method is also in `alsoDefensible`. A small content-only check is enough; there is no statistical computation engine in the MVP.

Design for a narrow phone screen first: one card above a vertical list of large method buttons, then a feedback panel in the same reading flow. On a wide screen, the buttons can sit beside the card. Never rely on color alone for status. Use semantic buttons, visible focus indicators, a clear selected state, accessible feedback announcements, and reduced-motion behavior. A click or keyboard action must cover every interaction even if optional dragging is added later.

No personal information is needed. Keep current-round state in memory; an optional local-only preference can remember whether a learner has seen the introduction. Do not send responses to an analytics service in the MVP.

## Netlify deployment

Build the app in its own repository or a clearly isolated app directory. For a root-level Vite app, use a `build` script that runs `vite build`, configure Netlify's build command as `npm run build`, and publish `dist`. Netlify documents those settings for Vite. [Netlify Vite deployment guide](https://docs.netlify.com/build/frameworks/framework-setup-guides/vite/)

Connect the repository to Netlify for deploy previews and production deploys. The MVP uses only `/`, so it needs no server routes or SPA fallback rule. If later versions add browser-history routes, configure a rewrite to serve `index.html` for those routes. Do not add Netlify Functions, Identity, or a database for the matching exercise. [Netlify guidance for Git-linked projects](https://docs.netlify.com/manage/projects/add-new-project/)

Before publishing, run the production build locally, open the generated site, and complete a full round with keyboard and touch-sized viewport checks. After the first Netlify deploy, verify the public URL on a phone and desktop browser and make sure a fresh visit loads the same authored deck.

## Delivery sequence and acceptance criteria

1. **Content review:** Finalize the ten cards, answer rules, and method definitions with an introductory statistics instructor. Resolve any card for which two methods are equally good under the stated question.
2. **Static app:** Build the single-page interface, local content module, selection and feedback flow, and round summary.
3. **Usability pass:** Ask a few target students to complete a round while explaining their choices. Revise wording when they infer missing data structure or misunderstand a variable's role.
4. **Launch:** Check the production build, keyboard flow, mobile layout, and Netlify deployment; then publish the reviewed deck.

The MVP is done when every card can be completed without dragging or a pointer; all ten cards have reviewed explanations and a condition-to-check note; repeated method labels work correctly; defensible alternatives receive fair feedback; a full round and retry work without a network connection after the page loads; and the Netlify-hosted build behaves like the local production build.

## Later extensions

Add contrast pairs that change **one** feature at a time, such as independent versus paired measurements, continuous versus binary outcome, or association versus adjusted modeling. This would directly show learners why the method changes. Further decks could introduce clustered data, nonparametric alternatives, time-to-event outcomes, and the point at which a statistician or instructor should review the analysis plan.
