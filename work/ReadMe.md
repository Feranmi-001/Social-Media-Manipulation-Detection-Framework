# Coordinated Social-Media Manipulation Detection: Project Notes

## 1. Pipeline
Dataset files -> Spark (PySpark) -> Parquet -> GraphFrames (graph, label
propagation) -> label-free coordination score -> MongoDB (twibot database)
-> evaluation and charts. Everything ran in Docker (Spark 3.5.3 + MongoDB 7)
on a laptop.

## 2. Data
- Files used: edge.csv, label.csv, split.csv, user.json (layout matches
  TwiBot-22; CONFIRM the exact dataset version with the supervisor).
- Not used: tweet files (not available), list.json, hashtag.json.
- Edge relations found: following, followers, post, pinned, own.
- Graph built from follow links only, between labeled users:
  693,761 users, 3,332,811 edges, 81,432 bots (11.7%).
- Analysis core: the 50,000 highest-degree users, 1,546,116 edges,
  3,118 bots (6.2%).

## 3. Method
- Community detection: GraphFrames label propagation (5 iterations),
  125 communities; the largest has 21,312 users.
- Per-community measures: size, density, bot ratio, lift (bot ratio / 0.062).
- Label-free signals from user.json: share of members created in the same
  month (burst_share), share with empty bios, median following/followers
  ratio.
- Coordination score = 0.30*burst_share + 0.30*empty_bio_share
  + 0.20*density + 0.20*follow_ratio (each converted to a percentile rank).
  Only communities with at least 50 members and at least one internal link
  per member were scored: 16 communities.

## 4. Results
- The four communities with the highest scores that were also bot-enriched
  (IDs ending 0502, 5336, 0164, 8645) have 2.0-2.9x the core's bot rate
  and 3-5x the baseline share of empty bios.
- Spearman(score, lift) across the 16 communities = 0.61.
- Community 77309411673 (ends 1673) has a high creation burst but lift 0.75:
  an open case (human cohort, or coordination the labels miss).
- Dense communities are not necessarily bot groups: several of the densest
  are mostly human.
- User-level evaluation (official split, only users in scored communities):
  test AUC 0.529 (train 0.631, val 0.495). Even the label-based community
  lift gives test AUC 0.531. Conclusion: the score flags suspicious
  communities for review but is NOT a reliable per-account bot detector.

## 5. Limitations
1. The graph was built from the first 5,000,000 rows of edge.csv, not the
   whole 6.6 GB file, because of laptop memory limits.
2. No tweet data, so no posting-time synchronization or text similarity.
3. The 50k core favors popular accounts (6.2% bots vs 11.7% overall).
4. Split class balance is very uneven (train 1.5% bots, val 25%, test 22% in
   the evaluated users), so AUCs across splits are not comparable.
5. Only 16 communities were scored; the score weights were chosen after
   viewing the community table, so results are indicative, not conclusive.
6. GraphFrames was used instead of Scala GraphX (no Python API); label
   propagation can give different community IDs on each run, so the saved
   result is fixed.

## 6. Future work
Add tweet-based signals (timing, shared hashtags and URLs), mutual-follow
graphs, Louvain/Leiden communities, and a per-user classifier.