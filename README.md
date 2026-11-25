<a id="top"></a>

<div align="center">

# **YouTube Insight: Multilingual Sentiment & Topic Analysis**

<b><ins>An NLP pipeline that collects YouTube comments, samples the most engaged ones, and summarizes each video's audience sentiment, keywords, topics and languages.</ins></b>

</div>

<br>

<details open>
<summary><b>Table of Contents</b></summary>

1. [Project Overview](#project-overview)
2. [Tech Stack](#tech-stack)
3. [How It Works](#how-it-works)
4. [Skills Learned](#skills-learned)
5. [Challenges and Solutions](#challenges-and-solutions)
6. [Project Structure](#project-structure)
7. [Extra Things](#extra-things)
8. [Credits and Contact](#credits-and-contact)

</details>

---

## <ins>Project Overview</ins>

Give the pipeline a list of YouTube links, and it fetches up to 5,000 comments per video, samples the most engaged ones, and returns an average sentiment score from 1 to 5, the top keywords, the discussion topics, and the language mix of the audience, all in one table.

### ▸ Problem
Popular YouTube videos get thousands of comments in many languages. Reading them all to understand how an audience feels, what they talk about, and where they come from isn't realistic for creators or brands. This project automates that: it turns a video's comment section into a short, structured summary.

### ▸ Objectives
- *Collect comments and video details* through the YouTube Data API, beyond the 100-comment limit per request.
- *Sample comments efficiently*, favoring the most engaged ones while keeping a random share for balance.
- *Score sentiment in many languages* and weight it by likes and replies.
- *Extract keywords and topics* and detect the languages used in the comments.
- *Return one clear table per batch of videos* that can be visualized or exported.

### ▸ Context
A personal project I built on my own in September 2025, as groundwork for a possible Chrome extension that shows these insights directly on YouTube.

<p align="right">(<a href="#top">back to top</a>)</p>

---

## <ins>Tech Stack</ins>

> The tools I **used** and what I used them **for**.

| Category | Tool | What I Used It For |
|:---|:---|:---|
| **Language** | `Python` | The full pipeline, from data collection to the summary table |
| **Data Processing** | `pandas`, `NumPy` | Storing comments, sampling, weighted averages and the final summary table |
| **AI / ML** | `PyTorch`, `Transformers` (Hugging Face), `sentence-transformers`, `BERTopic` | Running the multilingual sentiment model on GPU, creating multilingual embeddings and grouping comments into topics |
| **Libraries** | `google-api-python-client`, `langdetect`, `multi-rake`, `emoji`, `python-dotenv` | Calling the YouTube Data API v3, detecting comment languages, extracting keywords per language, removing emojis, and loading the API key |
| **Tools** | Jupyter | Building and running the pipeline in one notebook |

<p align="right">(<a href="#top">back to top</a>)</p>

---

## <ins>How It Works</ins>

> Fetch comments → sample the most engaged ones → clean → sentiment, keywords, topics and languages → one summary row per video.

```
YouTube URLs → Fetch Comments + Metadata → Biased Sampling → Cleaning → Sentiment / Keywords / Topics / Languages → Summary Table (CSV)
```

1. **Fetch.** For each video link, the pipeline extracts the video ID and gets the title, channel, publish date, views and likes. Comments come in pages of 100, so it keeps requesting the next page, with a 1-second pause between calls, until it reaches **5,000 comments** or runs out.
2. **Sample.** Running transformer models on thousands of comments is slow, so the pipeline takes about **30%** of the comments, with a minimum of 200 and a maximum of 2,000. The sample is split evenly into four groups: **most liked**, **most replied**, **most recent**, and a **random** share of the rest. Highly engaged comments are more likely to be included because they often show the opinions that resonated most.
3. **Clean.** Remove line breaks, HTML, links, emojis, years and long numbers, and symbols, while keeping accented letters for languages like Spanish and French. Then convert to lowercase.
4. **Analyze.**
   - **Language:** `langdetect` labels each comment with its language.
   - **Sentiment:** `tabularisai/multilingual-sentiment-analysis` scores each comment in batches on the GPU. The class probabilities are converted into one score from **1 (very negative) to 5 (very positive)**.
   - **Weighted sentiment:** each comment's weight is `1 + log(1 + likes) + log(1 + replies)`. The logarithm stops a single viral comment from dominating the average.
   - **Keywords:** RAKE extracts the top phrases from each comment, using the comment's own language for 12 priority languages (English, Hindi, Spanish, Russian, Vietnamese, German, French, Portuguese, Indonesian, Polish, Tagalog, and Turkish). The 10 most frequent phrases per video are kept.
   - **Topics:** `BERTopic` with `paraphrase-multilingual-MiniLM-L12-v2` embeddings groups comments into topics and returns the 5 largest, with 5 words each.
5. **Summarize.** Each video becomes one row: title, channel, views, likes, sample size, unweighted and weighted sentiment, top languages, keywords and topics. The table is saved as `youtube_summary_tabularis.csv`.

<p align="right">(<a href="#top">back to top</a>)</p>

---

## <ins>Skills Learned</ins>

> What this project taught me, technically and beyond.

- **Working with the YouTube Data API:** Paging through results with `nextPageToken`, adding a pause between calls, and handling errors so one failing video doesn't stop the batch.
- **Designing a sampling strategy:** Balancing processing cost and representativeness by mixing most liked, most replied, most recent and random comments.
- **Cleaning multilingual text:** Removing noise like links, HTML and emojis without stripping the accented letters other languages need.
- **Running transformer models:** Loading a Hugging Face model, running it in batches on the GPU, and turning class probabilities into one continuous score.
- **Topic modeling with embeddings:** Using BERTopic with multilingual sentence embeddings to group comments into themes.
- **Multilingual keyword extraction:** Choosing a RAKE model per language, with English as the fallback.
- **Weighted aggregation:** Using log-scaled engagement weights so popular comments count more without one comment dominating.
- **Critical evaluation of NLP output:** Reading the results closely to find where each model breaks down, and planning concrete fixes.

<p align="right">(<a href="#top">back to top</a>)</p>

---

## <ins>Challenges and Solutions</ins>

> What went wrong, how I fixed it, and what I took away from the project.

Sentiment scores stuck near neutral. Every video scored between 2.5 and 3.5, even a video as widely loved as *Despacito* (3.11). Two things cause this. The sentiment model is general-purpose and wasn't trained on YouTube-style comments full of sarcasm, memes, and slang. And the cleaning step removes emojis, which are often the clearest sign of emotion in a comment. A model trained on social media text, and converting emojis into words instead of deleting them, should both make the scores more decisive.

Topics full of filler words. Many topics were made of words like *the, and, to, is*, and one topic for *Despacito* contained almost no real words at all. BERTopic was grouping comments well, but describing each group with the most common words, including stopwords. Giving BERTopic a vectorizer that removes stopwords for each language, or using a keyword-based topic representation, would make the topics readable. Lowering `min_topic_size` from 5 to 3 should also give tighter clusters.

Language detection on short comments. `langdetect` labeled short English comments as Afrikaans, Welsh, Somali or Norwegian, and marked 118 *Despacito* comments as *unknown*, many of them probably emoji-only. Short texts don't contain enough letters for reliable detection. A stronger language identification model, such as fastText, and skipping very short comments would make the language mix more accurate.

<p align="right">(<a href="#top">back to top</a>)</p>

---

## <ins>Project Structure</ins>

```
YouTube-Insight-Multilingual-Sentiment-Topic-Analysis-/
├── Youtube Extension Project_PUBLIC.ipynb   # the full pipeline
└── README.md
```

<p align="right">(<a href="#top">back to top</a>)</p>

---

## <ins>Extra Things</ins>

<details>
<summary><b>Results and Key Findings</b></summary>
<br>

> **Headline:** Keywords captured each video's conversation much better than topics. For example, *medium rare*, *judges* and *steak* for the MasterChef video, and *half reps* and *spotter* for the gym video.

| Video | Comments Fetched | Sampled | Sentiment | Weighted | Top Languages |
|:---|:---:|:---:|:---:|:---:|:---|
| *Despacito*: Luis Fonsi ft. Daddy Yankee | 5,000 | 1,377 | 3.11 | 3.12 | en 600, es 166, de 49, pt 46 |
| *About You*: The 1975 | 3,097 | 818 | 3.23 | 3.34 | en 633, id 51 |
| *Perfect Medium Rare Steak*: MasterChef World | 2,017 | 526 | 2.76 | 2.88 | en 461 |
| *Data Structures and Algorithms in 15 Minutes* | 1,061 | 282 | 3.06 | 3.07 | en 256 |
| *Sam Sulek Throws 315* | 530 | 174 | 2.53 | 2.50 | en 158 |
| *Data Structures, Explained Simply*: Nic Barker | 122 | 122 | 3.45 | 3.38 | en 116 |

1. **Keywords work well:** RAKE picked out specific, meaningful phrases for every video.
2. **Weighting changes little:** the weighted and unweighted scores differ by at most 0.12. For *About You* and the MasterChef video, the most-liked comments were slightly more positive.
3. **The audience mix shows up clearly:** *Despacito* had the most multilingual comments, with large Spanish, German and Portuguese groups, and *About You* had a notable Indonesian audience.
4. **Limits:** sentiment clusters near neutral and topics include filler words. See [Challenges and Solutions](#challenges-and-solutions).

</details>

<details>
<summary><b>Model Details</b></summary>
<br>

- **Sentiment:** `tabularisai/multilingual-sentiment-analysis`, batch size 32, maximum 512 tokens; probabilities are converted to an expected score from 1 to 5, and comments longer than 512 tokens are flagged as truncated
- **Topics:** `BERTopic` with `paraphrase-multilingual-MiniLM-L12-v2` embeddings, `min_topic_size=5`, 5 words per topic, top 5 topics per video
- **Keywords:** `multi-rake`, one model per priority language with English as the fallback, top 5 phrases per comment, top 10 per video
- **Language detection:** `langdetect` with a fixed seed for repeatable results
- **Sampling:** 30% of fetched comments, minimum 200 and maximum 2,000, split evenly across most liked, most replied, most recent and random
- **Hardware:** runs on GPU with CUDA when available, otherwise CPU

</details>

<details>
<summary><b>Planned Improvements</b></summary>
<br>

- **Better sentiment:** switch to a model trained on social media text, such as `cardiffnlp/twitter-roberta-base-sentiment-latest`, combined with a multilingual model like XLM-RoBERTa for non-English comments, and keep emojis as words instead of removing them.
- **Better keywords:** combine **KeyBERT**, which uses meaning instead of word frequency, with RAKE for multilingual support, and show only the top 3 keywords per video for a cleaner display.
- **Better topics:** remove stopwords before describing topics, and lower `min_topic_size` to 3 for tighter clusters.
- **Better language detection:** use fastText language identification and skip very short comments.
- **Tuned weighting:** experiment with the log scaling of likes and replies to give highly engaged comments more influence.
- **Chrome extension:** show the insights directly on YouTube while browsing.
- **Scale:** use asynchronous requests and caching to analyze many more videos efficiently.

</details>

<p align="right">(<a href="#top">back to top</a>)</p>

---

## <ins>Credits and Contact</ins>

**Acknowledgements:** Built on the open-source **tabularisai/multilingual-sentiment-analysis** model, **sentence-transformers** and **BERTopic**, and on data from the **YouTube Data API v3**.

**License:** Personal use only.

**Author:** **Hansel Sebastian**

**Contact:** [GitHub](https://github.com/hanhansel) · [LinkedIn](https://linkedin.com/in/hansel-sebastian-5a19b8245)

<p align="right">(<a href="#top">back to top</a>)</p>
