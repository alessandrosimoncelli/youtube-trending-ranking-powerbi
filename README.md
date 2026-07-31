# YouTube Trending Video Ranking Algorithm

A Power BI ranking system that recommends the top trending YouTube videos in Canada (Nov 2017 to Jun 2018) using a composite, engagement-weighted score. The goal was to prevent raw view counts from dominating the ranking, similar to how recommender systems at YouTube, Netflix, and Spotify balance reach against actual audience engagement.

Built for Business Intelligence (DAT-8564), Hult International Business School, Spring 2026.

## The Problem

YouTube engagement metrics operate on very different scales. In this dataset, total Views (about 47 billion) outweighs total Dislikes (about 80 million) by nearly three orders of magnitude. If combined directly into a weighted sum, Views ends up dominating the score no matter what weights are assigned. A video with 10 million views and zero comments would outrank one with 500,000 views and 50,000 comments, which defeats the point of building a multi-dimensional ranking system in the first place.

## Methodology

### Normalization: four methods tested, one selected

| Method | Formula | Verdict |
|---|---|---|
| Simple Min-Max | `(X - Min) / (Max - Min)` | Rejected. Compresses about 95% of videos into a 0.00 to 0.05 range, so only the viral outliers are actually distinguishable |
| Z-Score | `(X - Mean) / StdDev` | Rejected. Output is unbounded, and the heavy right skew of engagement data pulls the mean upward, producing mostly negative values with no fixed scale to compare across metrics |
| Box-Cox | `(X^λ - 1) / λ` | Rejected. Requires estimating a lambda parameter, doesn't naturally bound output to [0,1], and needs a constant added to handle zero values. Too much added complexity for no real benefit here |
| Logarithmic (LOG10) | `LOG10(X + 1) / MAX(LOG10(X + 1))` | Selected. Compresses extreme values at the top while expanding the lower and middle range, bounded to [0,1], keeps ordinal ranking intact, no parameter estimation needed |

All four methods are built as separate DAX measures and compared on a dedicated dashboard page, with scatter plots and a statistical comparison table, so the choice is something you can check rather than take on faith.

### Ranking formula

```
Ranking Score = (Normalized Views x 10%)
              + (Normalized Engagement x 80%)
              + (Normalized Days Trending x 10%)

Normalized Engagement = (Normalized Comments x 50%)
                       + (Normalized Dislikes x 30%)
                       + (Normalized Likes x 20%)
```

Engagement carries most of the weight (80%) because comments, likes, and dislikes all require a deliberate user action, which makes them stronger quality signals than a passive view. Views are kept low on purpose (10%), since a high view count can come from clickbait, autoplay, or algorithmic promotion rather than actual content quality. Weighting views heavily would basically just recreate a sort-by-views list.

Dislikes are weighted above likes (30% vs 20%) because a dislike still takes conscious effort to register. Controversial videos tend to be highly engaging, so the algorithm treats dislikes as a real signal instead of a penalty.

### Dynamic weight redistribution

Some videos have `comments_disabled` or `ratings_disabled` set to true. A zero in these fields doesn't mean the video got no engagement, it means that metric was never available. The Normalized Engagement measure checks for this and redistributes weight across whatever metrics are actually available. If ratings are disabled, for example, Comments picks up 100% of the engagement weight instead of 50%. This keeps videos from being unfairly punished for missing data rather than poor performance.

## Data Pipeline (ETL)

- Source: `CAvideos.csv` (one row per video per trending date, about 40,000 rows) plus `CA_category_id.json` (category lookup)
- Fixed a Power Query parsing issue caused by embedded line breaks in quoted fields
- Rebuilt `trending_date` from the original `YY.DD.MM` text format into a proper `YYYY-MM-DD` date type
- Manually checked and corrected every column's data type instead of trusting Power BI's auto-detection
- Filtered out an orphan category (`category_id = 29`, missing from the lookup table), blank categories, and removed or errored videos
- Built a `DaysTrending_Lookup` table and a separate `DateTable` (set as the official date table) so time-based logic stays out of the fact table
- Used a single-direction relationship from Categories to Videos to keep DAX filter behavior predictable

The full rationale behind each transformation is documented in the methodology report.

## Repository Contents

| File | Description |
|---|---|
| `YouTube_Ranking_Dashboard_Canada-2.pbix` | The Power BI file itself: data model, all DAX measures, and both dashboard pages. Open this in Power BI Desktop to explore or edit the ranking logic directly |
| `YoutubeRanking_Step-by-Step_Methodology_Guide.pdf` | Written report covering the ETL process, the comparison of all four normalization methods, the reasoning behind every weight in the ranking formula, a walkthrough of the dashboard, and the final insights drawn from the data |
| `README.md` | This file, a short overview of the project for anyone browsing the repo |

## Dashboard

Page 1, Leaderboard: shows the top 5 recommended videos for whatever date and category is selected, with thumbnail, ranking score, and a breakdown of each normalized metric. Slicers let you filter by category, date (year, month, day), and weekday vs weekend.

Page 2, Statistical Methods Comparison: shows how the ranking output changes across all four normalization methods side by side, including scatter plots comparing a views-only ranking against the LOG-based ranking, and Z-score behavior versus LOG behavior under outliers.

## Key Insights

1. The algorithm-based ranking and a views-only ranking are positively correlated but noticeably diverge for a good number of videos, meaning the engagement-weighted score is picking up on something a simple views sort would miss.
2. The Z-score distribution showed a handful of videos with extreme values, confirming how sensitive it is to outliers. This is what pushed the choice toward LOG normalization, and the Simple vs LOG scatter plot shows a clean S-curve as a result, meaning scores end up spread more evenly across the dataset.
3. The dynamic weight redistribution shows up clearly in practice. For example, the video "To Our Daughter" ranks third even though its Likes, Dislikes, and Comments all show 0.00, because ratings and comments were disabled for that video and the algorithm correctly shifted the weight elsewhere instead of penalizing it.
4. There's a real difference between weekend and weekday trending content. Movies and Pets & Animals rank higher on weekends, while Science & Technology and How-to & Style do better on weekdays. This suggests day of week could be a useful signal for recommender systems more generally.

## Tools

Power BI Desktop, DAX, Power Query (M)

## Data Source

[YouTube Trending Video Dataset (Canada region)](https://www.kaggle.com/datasnaek/youtube-new), Nov 2017 to Jun 2018

## Team

Alessandro Simoncelli, Akihiro Taguchi, Carlotta Signori, Lindokuhle Sikhondze
Hult International Business School, DAT-8564, Spring 2026

## Reference

Milli, S., Carroll, M., Wang, Y., Pandey, S., Zhao, S., & Dragan, A. D. (2023). Engagement, user satisfaction, and the amplification of divisive content on social media. Proceedings of the ACM Conference on Fairness, Accountability, and Transparency (FAccT).
