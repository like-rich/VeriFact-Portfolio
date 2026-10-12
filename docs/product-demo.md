# Product demo and capture notes

[中文首页](../README.md) · [English overview](../README_EN.md)

## What the images show

The screenshots were captured on 12 October 2026 from the original project's React/Vite frontend running locally with `VITE_USE_MOCKS=true`. The portfolio owner confirmed that the interface screenshots may be published.

These are actual rendered interfaces, not redesigned mockups. They use the project's built-in demonstration fixtures rather than real user accounts, private records, or live LLM responses. Displayed claims are inputs to the fact-checking and review interfaces, not statements endorsed by this portfolio.

| Capture | Visible capability |
| --- | --- |
| [Landing page](../assets/landing-page.png) | Product introduction and verification entry points |
| [Claim input](../assets/claim-input.png) | Text composer with a demonstration claim |
| [Verification report](../assets/verification-result.png) | Verdict, confidence, concept-level evidence, reasoning, and source links |
| [Reviewer queue](../assets/reviewer-queue.png) | Low-confidence, random-audit, and source-conflict routing |
| [Review workbench](../assets/reviewer-workbench.png) | AI output alongside human verdict, error-category, and notes controls |
| [Analytics metric area](../assets/analytics.png) | Agreement, accuracy, false-confidence, and review summary cards |

## GIF walkthrough

[The 30-second GIF](../assets/demo.gif) contains six actual interface captures, each displayed for five seconds: landing page, input, report, reviewer queue, workbench, and analytics. It is a screenshot sequence showing the UI workflow, not a continuous recording or a timing benchmark. The reviewer queue uses separate built-in sample tasks; it does not show that the displayed 96%-confidence claim was routed into the queue.

## Scope and limits

- The text submission was exercised through the frontend's mock verification path. It demonstrates UI integration, not a live retrieval or model evaluation.
- Report source links are fixture links, not citations retrieved during this capture session.
- Values such as 96% confidence and 83.3% accuracy are mock values. They must not be reused as measured project achievements.
- The review workbench was opened without submitting a human verdict.
- Captures are cropped to remove browser chrome and unrelated tabs, resized, and stripped of embedded metadata. The analytics image focuses on the metric-card area. Interface content was not rewritten.
- The GIF adds a caption explicitly identifying mock data. No private course source code is included in these assets.

个人贡献见 [My Contributions](my-contributions.md)；截图展示的是团队产品，不能据此认定所有界面与模块均由个人独立实现。
