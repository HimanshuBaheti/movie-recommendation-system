# 🎬 Movie Recommendation System

A **content-based movie recommendation web application** built with **Python, Flask, pandas, and scikit-learn**. It uses movie metadata to find films similar to a selected title, ranking candidates by **cosine similarity** over bag-of-words features.

The application serves movie-search and recommendation pages with HTML/CSS/JavaScript, shows film details, and retrieves additional poster and trailer information from **The Movie Database (TMDB) API**.

> **Note:** This repository contains an older Flask implementation. The recommender logic is present, but some frontend AJAX endpoints are not implemented and the application needs cleanup before reliable deployment. See [Known limitations](#known-limitations).

## ✨ Features

- **Content-based recommendations:** Retrieves the **10 most similar movies** for a matching title using vectorized movie metadata.
- **Metadata similarity:** Uses a preprocessed text field named `soup` from the included TMDB-based dataset.
- **Movie search interface:** Submit a title using the Flask-rendered web UI, with autocomplete suggestions.
- **Recommendation cards:** Templates display movie titles, release dates, ratings, and descriptions; the home page shows a subset of suggestions.
- **Posters and trailers:** Frontend templates make TMDB API requests for posters and movie videos.
- **Random starting recommendations:** The homepage selects a movie from `movieR.csv` and generates recommendations from it.
- **Search history file:** Submitted movie names are appended to `movieR.csv`.

## 🛠️ Tech stack

| Component | Technology |
| --- | --- |
| Language | Python |
| Web framework | Flask |
| Data processing | pandas |
| Feature extraction | scikit-learn `CountVectorizer` |
| Similarity measure | Cosine similarity |
| Frontend | HTML, CSS, JavaScript, jQuery |
| Movie metadata | TMDB-derived CSV dataset |
| Supplementary content | TMDB API |
| Deployment configuration | Gunicorn, `Procfile` |

## 🧠 How the recommendation engine works

This is a **content-based filtering** system: it recommends similar films based on metadata, **not** user ratings, collaborative filtering, TF-IDF, or a trained deep-learning model.

```text
TMDB-based movie dataset
           |
           v
Preprocessed "soup" column
           |
           v
CountVectorizer(stop_words="english")
           |
           v
Sparse bag-of-words feature matrix
           |
           v
Cosine similarity between movie vectors
           |
           v
Find the requested movie by title
           |
           v
Sort similar films by descending score
           |
           v
Return the 10 closest titles and metadata
           |
           v
Render recommendation cards in Flask templates
```

### Algorithm

The implementation in `app.py` converts the dataset's `soup` column into a bag-of-words representation and ranks films by cosine similarity:

```python
count = CountVectorizer(stop_words="english")
count_matrix = count.fit_transform(df2["soup"])

def get_recommendations(title):
    cosine_sim = cosine_similarity(count_matrix, count_matrix)
    idx = indices[title]

    sim_scores = list(enumerate(cosine_sim[idx]))
    sim_scores = sorted(sim_scores, key=lambda x: x[1], reverse=True)
    sim_scores = sim_scores[1:11]

    movie_indices = [i[0] for i in sim_scores]
    return df2.iloc[movie_indices]
```

For feature vectors `A` and `B`, cosine similarity is defined as:

```text
cosine_similarity(A, B) = (A · B) / (||A|| × ||B||)
```

The implementation selects the ten highest-ranked other movies. The existing `get_recommendations()` also assembles fields for titles, release dates, ratings, overviews, keywords, and movie IDs.

**Performance note:** `app.py` computes a full pairwise cosine similarity matrix during startup and again for each recommendation request. This can consume substantial time and memory for a large dataset; a more scalable approach would compute similarity only between the requested movie and other vectors.

## 📁 Repository structure

```text
movie-recommendation-system/
├── app.py                   # Flask routes and recommendation logic
├── model/
│   └── tmdb.csv             # Movie feature data used by recommender
├── tmdb.csv                 # Movie titles used for suggestions
├── movieR.csv               # Seed titles and appended searches
├── templates/
│   ├── index.html           # Home page and search form
│   ├── positive.html        # Search-results / recommendation page
│   ├── negative.html        # Movie-not-found page
│   └── test.html            # Additional template
├── static/
│   ├── autocomplete.js      # Search autocomplete
│   ├── recommend.js         # TMDB-related client-side requests
│   ├── script.js            # Additional UI interactions
│   └── style.css            # Styling
├── elements/                # UI assets and example components
├── requirements.txt         # Python dependencies
├── Procfile                 # Gunicorn deployment entrypoint
└── README.md
```

## 🚀 Run locally

### Prerequisites

- Python and `pip`
- Internet access for externally hosted frontend assets and TMDB API calls
- Sufficient RAM to build the pairwise similarity matrix

### 1. Clone the repository

```bash
git clone https://github.com/HimanshuBaheti/movie-recommendation-system.git
cd movie-recommendation-system
```

### 2. Create a virtual environment

```bash
python -m venv .venv
```

Activate it:

**macOS/Linux**

```bash
source .venv/bin/activate
```

**Windows (PowerShell)**

```powershell
.venv\Scripts\Activate.ps1
```

### 3. Install dependencies

```bash
pip install flask gunicorn pandas scikit-learn
```

> The repository's `requirements.txt` specifies `sklearn`, an outdated package name. For a fresh environment, install `scikit-learn` as shown above. The app imports `sklearn` from that distribution.

### 4. Launch the Flask server

```bash
python app.py
```

By default, Flask serves the development application at **http://127.0.0.1:5000/**.

Alternatively, on platforms supporting Gunicorn:

```bash
gunicorn app:app
```

The repository's `Procfile` uses this Gunicorn entrypoint.

### 5. Try a recommendation

Open the homepage, type a movie title found in the dataset, and submit the form. The `POST /positive` route looks up the title and renders a results page with the most similar movies. Titles not found in the dataset use `negative.html`.

**Important:** The implementation expects titles to match its dataset after applying Python's `.title()` transformation. It does not implement robust fuzzy title matching on the server.

## 🌐 Flask routes

| Method | Route | Function |
| --- | --- | --- |
| `GET` | `/` | Homepage with recommendations for a seed movie |
| `GET` | `/index` | Alias of the homepage |
| `GET` | `/positive` | Basic results-page route (no recommendation context provided) |
| `POST` | `/positive` | Look up submitted movie and render results or a not-found page |

The browser code in `static/recommend.js` also references `POST /similarity` and `POST /recommend`. Those endpoints are **not defined** in the current Flask server.

## 📊 Dataset

The repository includes two TMDB-derived CSV files:

- `model/tmdb.csv` — read by the recommendation engine and expected to include `soup`, `title`, `release_date`, `vote_average`, `overview`, `keywords`, and `id`.
- `tmdb.csv` — read to populate movie-title suggestions.

The `soup` column contains preprocessed movie content used to compare films. The recommender builds a vocabulary with English stop words removed.

The application is **not** trained using personal viewing histories or explicit user feedback, and recommendations represent metadata similarity rather than predicted enjoyment.

## ⚠️ Known limitations

These are observations from the committed code, not features promised for the application:

1. **Missing AJAX routes:** `static/recommend.js` calls `/similarity` and `/recommend`, but `app.py` does not define either route. Some frontend interactions will fail.
2. **Large similarity matrix:** Full all-pairs cosine similarity is calculated at startup and per request, which is expensive in both memory and computation.
3. **TMDB API key management:** Existing templates contain a hardcoded API key, while `static/recommend.js` contains the placeholder `YOUR_API_KEY`. API credentials should be removed from tracked frontend files and managed securely. Any exposed key should be rotated.
4. **Strict title matching:** The server relies on exact lookup after title-casing input. Misspellings, naming variations, and duplicate titles are not handled reliably.
5. **Mutable search CSV:** `movieR.csv` is appended to during normal requests. This can cause concurrency and persistence issues in production environments.
6. **Deployment link may be outdated:** The older Heroku URL from the original README is preserved below for reference but has not been verified as currently operational.
7. **No automated tests or evaluation report:** The repository does not include measurable recommendation-quality metrics or an automated test suite.
8. **Dependency compatibility:** Existing dependencies are unpinned, and the `sklearn` requirement should be updated to `scikit-learn`.

## 🔧 Suggested improvements

- Replace the full similarity matrix with on-demand cosine similarity against a single query vector.
- Consolidate the search UI into supported Flask routes and remove outdated AJAX calls.
- Improve normalization and approximate matching of movie titles.
- Keep API secrets out of tracked code and configure them securely.
- Persist search history in a database rather than appending to a CSV on every request.
- Add unit tests, recommendation evaluation, and graceful fallbacks for missing TMDB responses.
- Pin compatible dependencies and improve deployment reproducibility.

## 🔗 Previous deployment

The original README linked to:

[https://therecommender1.herokuapp.com/](https://therecommender1.herokuapp.com/)

This is a **historical deployment URL**; its current availability is not guaranteed.

## 👤 Repository owner

[Himanshu Baheti](https://github.com/HimanshuBaheti)

---

*Built as a hands-on demonstration of content-based movie retrieval, text vectorization, cosine similarity, and Flask web development.*
