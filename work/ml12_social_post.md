# ML-12 — Social Post

I built a content-refresh prioritization workflow using anonymized FlyRank search and engagement data covering 30,000 webpages across 32 clients.

I compared the ML-07 rule-based baseline with a Logistic Regression model using client-grouped validation. On the recorded evaluation, Precision@20 improved from 0.55 to 1.00 and Precision@50 improved from 0.48 to 1.00.

The result is a decision-support ranking that can help content teams decide which webpages deserve earlier human review. The model is not treated as a causal guarantee that refreshing a page will improve future performance.

**Deployed paper:** https://aryav11.github.io/flyrank-ml-internship/