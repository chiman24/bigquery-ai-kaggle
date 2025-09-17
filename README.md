# From Theme to Setlist

Line-level Semantic Lyric Finder powered by Google Cloud BigQuery, Vertex AI embeddings, and vector search. The accompanying notebook (`semantic-detective-song-finder.ipynb`) walks through loading worship song lyrics, generating embeddings, and surfacing the lyric lines that best match a theme.

## What You Build
- BigQuery dataset with songs, line-level lyrics, and embedding tables sourced from a Google Cloud Storage bucket
- Vertex AI remote model bound to BigQuery ML for `ML.GENERATE_EMBEDDING`
- Reproducible vector search workflow that retrieves the top lyric lines for any theme and aggregates the best-matching line per song
- Lightweight evaluation loop that scores search quality with NDCG and MRR against a hand-labelled set

## Prerequisites
- Google Cloud project with BigQuery, Vertex AI, and Cloud Storage APIs enabled
- BigQuery connection that can call Vertex AI (`BQ_CONN`), e.g. `your-project.us.vertex_conn`
- GCS bucket that stores lyric files (`GCS_BUCKET`), one file per song, named `Song Title - Artist.txt`
- Ability to authenticate to GCP from the execution environment (Kaggle notebook, Cloud AI Platform Notebook, or local machine with `gcloud auth login`)

### Python packages
The first cell in the notebook installs everything that is needed:

```bash
pip install "google-cloud-bigquery[bqstorage,pandas]" google-cloud-bigquery-storage pyarrow
```

If you run the notebook outside Kaggle, make sure those packages are already available in your environment before executing the remaining cells.

## Notebook Flow
1. **Configure constants** – set `PROJECT_ID`, `LOCATION`, `DATASET`, `BQ_CONN`, `EMBED_MODEL`, `THEME_TEXT`, `TOP_K`, and `GCS_BUCKET` at the top of the notebook.
2. **Initialize clients** – create BigQuery and Cloud Storage clients with your active credentials.
3. **Dataset + tables** – create the target BigQuery dataset, plus managed tables (`songs`, `song_lines`) and their staging counterparts.
4. **Remote embedding model** – deploy a BigQuery remote model that fronts the Vertex AI text embedding endpoint (default `text-multilingual-embedding-002`).
5. **ETL from GCS** – read lyric text files, normalize line breaks, and load data into staging tables before MERGE-ing into the main tables.
6. **Generate embeddings** – call `ML.GENERATE_EMBEDDING` twice to produce song-level and line-level embeddings using document/query task types; persist results into `song_embeddings` and `song_line_embeddings`.
7. **Vector search** – embed the theme (`task_type='RETRIEVAL_QUERY'`) and call `VECTOR_SEARCH` on the line embeddings. The helper function surfaces each song’s best matching line.
8. **Evaluation** – compare ranked results against a labelled set using NDCG@k and MRR@k to gauge retrieval quality.

Each section includes markdown guidance and executable SQL/Python so you can run the pipeline top-to-bottom.

## Preparing Lyric Data
- Store raw lyrics as plain text files in your `GCS_BUCKET`.
- Use file names formatted as `Song Title - Artist.txt`; the notebook parses the title and artist from the path.
- Maintain one lyric line per line in the text file. Headings such as `Verse 1:` or `Chorus:` will be captured as the surrounding section metadata.

## Outputs
Running the notebook creates or updates the following BigQuery assets inside `${PROJECT_ID}.${DATASET}`:
- `songs` and `song_lines` tables (plus `_staging` companions)
- `song_embeddings` and `song_line_embeddings` tables containing vector columns (`song_vec`, `line_vec`)
- Optional evaluation table (commented) that stores search rankings for multiple themes
- Remote model `embed_text` that wraps the Vertex AI embedding endpoint

## How to Run
1. Open `semantic-detective-song-finder.ipynb` in your environment of choice.
2. Edit the configuration cell with your project IDs, dataset name, connection, and bucket.
3. Execute cells sequentially; the notebook prints progress for dataset/table creation, loads, embedding jobs, and searches.
4. Adjust `THEME_TEXT` or provide a list of themes when running the ranking/evaluation helpers to explore different setlists.

## Troubleshooting
- **Authentication errors** – confirm you have Application Default Credentials in the session (e.g. `gcloud auth application-default login` locally, or Kaggle’s GCP integration enabled).
- **Missing connection** – create the Vertex AI connection in BigQuery and grant the service account `roles/aiplatform.user` and `roles/storage.objectViewer` on the bucket.
- **Vector search quota** – ensure your project has BigQuery BI Engine reservations or vector search quota in the target region.
- **Staging tables** – rerun the load cells; they MERGE into production tables and clear the staging tables automatically.

## Next Steps
- Build a lightweight UI that calls the notebook logic via Cloud Functions or Cloud Run for on-demand setlist planning.
