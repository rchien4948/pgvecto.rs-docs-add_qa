# Sparse Vector Search

In the past, hybrid search combined two search methods: traditional keyword-based search and vector-based similarity search. However, sparse vector can act as a substitute for keyword search, unlocking the full potential of the data pipeline with pure vector search.

::: tip
For more information about keyword-based hybrid search, please refer to [Hybrid Search](./hybrid-search.md)
:::


This post explores how to use how [BGE-M3](https://github.com/FlagOpen/FlagEmbedding/tree/master/FlagEmbedding/BGE_M3) and `pgvecto.rs` to exploit the potential of hybrid search using both sparse and dense vectors.

Let's dive in.

## An Overview of Sparse and Dense Vector Embedding

Sparse vectors are high-dimensional vectors that contain few non-zero values. They are suitable for traditional information retrieval use cases. For example, a vector with 32,000 dimensions but only 2 non-zero elements is a typical sparse vector.
$$
S[1\times 32000]=\left[
 \begin{matrix}
   0 & 0 & 0.015 & 0 & 0 & 0 & 0 & \cdots & 0.543 & 0
  \end{matrix}
  \right]
$$


Dense vectors are embeddings from neural networks. They are generated by text embedding models and have most or all elements non-zero. They have fewer dimensions, such as 256 or 1536, much less than sparse vectors.
$$
D[1\times 256]=\left[
 \begin{matrix}
   0.342 & 1.774 & 0.087 & 0.321 & 0.664 & 0.870 & 0.001 & \cdots & 0.543 & 0.999
  \end{matrix}
  \right]
$$


Dense vectors and sparse vectors are both used to represent features of text chunks, but they have significant differences. A single dimension in a dense vector embedding does not mean anything, as it is too abstract to determine its meaning. However, when we take all the dimensions together, they provide the semantic meaning of the input text.

Sparse vectors have dimensions equal to the vocabulary size, with each element representing the weight of a word in the text.

Generally, words with higher weights indicate greater importance in the text. Sparse vectors simplify the analysis and visualization of the relationship between text and vocabulary. To visualize dense vectors, additional dimension reduction algorithms are necessary, such as:

- [Principal component analysis (PCA)](https://en.wikipedia.org/wiki/Principal_component_analysis)
- [T-distributed stochastic neighbor embedding (T-SNE)](https://en.wikipedia.org/wiki/T-distributed_stochastic_neighbor_embedding)
- [Uniform Manifold Approximation and Projection (UMAP)](https://arxiv.org/abs/1802.03426)

|                      | Dense Vector          | Sparse Vector               |
| -------------------- | --------------------- | --------------------------- |
| Dimension            | Hundreds to thousands | Tens of thousands           |
| Element Meaning      | Nothing               | Word Weight                 |
| Interpretability     | Hard                  | Easy                        |
| Usage                | Vector Distance       | Vector Distance / Visualize |
| Semantic sensitivity | High                  | Low                         |

Sparse vectors are generally less effective at capturing the relationships between words, particularly for synonyms and related terms, compared to dense vectors.

For instance, both `rocket` and `SPACE X` are related to space exploration, but Dense vectors are more likely to reveal similarities and assign a closer vector distance to corresponding texts.

## How to create a vector embedding?

A vector embedding is the internal representation of input data in deep learning models, also known as embedding models. Most embedding models, such as `text-embedding-3-small`, only output dense embeddings. 

[SPLADE](https://europe.naverlabs.com/research/computer-science/splade-a-sparse-bi-encoder-bert-based-model-achieves-effective-and-efficient-full-text-document-ranking/?utm_source=qdrant&utm_medium=website&utm_campaign=sparse-vectors&utm_content=article&utm_term=sparse-vectors) and [BGE-M3](https://arxiv.org/pdf/2402.03216.pdf) can generate sparse embeddings, sometimes called lexical weights. In this section, we will introduce the `BGE-M3` model for generating dense and sparse vectors.

`BGE-M3` is a multi-functionality model, which can simultaneously perform the three common retrieval functionalities of the embedding model: dense retrieval, multi-vector retrieval, and sparse retrieval. We will learn how to generate dense and sparse embeddings from it.

```python
# pip install -U FlagEmbedding

from FlagEmbedding import BGEM3FlagModel

model = BGEM3FlagModel("BAAI/bge-m3", use_fp16=True)
sentences = ["What is BM25?", "Definition of BM25"]

output = model.encode(sentences, return_dense=True, return_sparse=True)

# The dense embedding of the two texts above:
dense_embedding = output['dense_vecs']
# [[-0.02501617 -0.04525582 -0.01972856 ... -0.0099566   0.01677509
#   0.01292699]
# [-0.0106499  -0.04731942 -0.01477693 ... -0.0149255   0.01040097
#   0.00965083]

sparse_weight_id = output['lexical_weights']

# You can see the weight for each token:
# print(model.convert_id_to_token(sparse_weight_id))
# 
# [{'What': 0.10430284, 'is': 0.10090457, 'BM': 0.2635918, '25': 0.3382988, '?': 0.052101523}, 
# {'Definition': 0.14298248, 'of': 0.076763615, 'BM': 0.2577639, '25': 0.33806682}]
```

For all sentences and words, a text-word matrix was created:

| TEXT               | What | is   | BM   | 25   | ?    | Definition | of   | ...  |
| ------------------ | ---- | ---- | ---- | ---- | ---- | ---------- | ---- | ---- |
| What is BM25?      | 0.08 | 0.08 | 0.13 | 0.25 | 0.04 | 0          | 0    | ...  |
| Definition of BM25 | 0    | 0    | 0.25 | 0.33 | 0    | 0.14       | 0.06 | ...  |


The matrix is highly sparse, especially when all vocabularies are included. Most words only appear in a few sentences, resulting in non-zero weights.

By feeding text into sparse embedding models like `BGE-M3`, we can easily obtain sparse embeddings from it.

```python
# pip install -U numpy

import numpy as np

# Sparse vector embedding of text
sparse_indices = [[int(token_id) for token_id in text] for text in sparse_weight_id]
# [[4865, 83, 90017, 2588, 32], [155455, 111, 90017, 2588]]
sparse_values = [[float(text[token_id]) for token_id in text] for text in sparse_weight_id]
# [[0.10430284, 0.10090457, 0.2635918, 0.3382988, 0.052101523], [0.14298248, 0.076763615, 0.2577639, 0.33806682]]

# Sparse indexes must be ascending for each pgvecto.rs
for i in range(len(sparse_weight_id)):
    sparse_values[i] = [x for _, x in sorted(zip(sparse_indices[i], sparse_values[i]))]
    sparse_indices[i] = sorted(sparse_indices[i])
```

Now we have both the sparse and dense embeddings. It's time to insert them into the vector database.

## Storing vectors in pgvecto.rs

First, a deployed `pgvecto.rs` instance is required for vector storage and retrieval. This can be achieved by the official `pgvecto-rs` Docker image:

```bash
docker run \
  --name pgvecto-rs-demo \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  -d tensorchord/pgvecto-rs:pg16-v0.3.0
```

Now the only necessary dependency is installed. Let's create a table to store the vectors.

`pgvecto.rs` supports dense or sparse vectors with different data types:

- `vector` for dense vector
- `svector` for sparse vector

The `BGE-M3` model has a dense embedding dimension of 1024. The sparse embedding dimension is 250002, which matches the size of tokenizer vocabulary.

```python
print(len(model.tokenizer))
# 250002
```

In this example, we create a table named `documents` with four columns: an ID column(`id`), a text column(`text`) and two vector columns(`sparse` for sparse vector and `dense` for dense vector). 

```python
# pip install -U psycopg

import psycopg
URL = "postgresql://postgres:mysecretpassword@localhost:5432/postgres"

with psycopg.connect(URL) as conn:
    conn.execute("DROP TABLE IF EXISTS documents;")
    conn.execute(f"""
        CREATE TABLE documents (
            id SERIAL PRIMARY KEY, 
            text TEXT NOT NULL,
            dense vector(1024) NOT NULL,
            sparse svector({len(model.tokenizer)}) NOT NULL);
    """
    )
```

With the table created, we can now insert embeddings into the table and create indexes for vector columns.

```python
# pip install -U pgvecto_rs

import psycopg
from pgvecto_rs.psycopg import register_vector
URL = "postgresql://postgres:mysecretpassword@localhost:5432/postgres"

with psycopg.connect(URL) as conn:
    register_vector(conn)
    for text, dense, sparse_ind, sparse_val in zip(sentences, dense_embedding, sparse_indices, sparse_values):
        conn.execute(
            "INSERT INTO documents (text, dense, sparse) VALUES (%s, %s, to_svector(%s, %s::real[]::int[], %s::real[]));",
            (text, dense, len(model.tokenizer), sparse_ind, sparse_val),
        )
    conn.execute("""
        CREATE INDEX ON documents 
        USING vectors (sparse svector_dot_ops) 
        WITH (options = \"[indexing.hnsw]\");
    """)
    conn.execute("""
        CREATE INDEX ON documents 
        USING vectors (dense vector_l2_ops) 
        WITH (options = \"[indexing.hnsw]\");
    """)
```

For dense vectors, `vector_l2_ops` calculates the Squared Euclidean distance, which is the most commonly used distance metric. 
$$
D_{L2} = \Sigma (x_i - y_i) ^ 2
$$
For sparse vectors, `vector_dot_ops` calculates the dot product, which is a more efficient method.
$$
D_{dot} = - \Sigma x_iy_i
$$


For more information about index types in `pgvecto.rs`, please [check out our documentation](https://docs.pgvecto.rs/getting-started/overview.html).

## Retrieving and Mixing Results

With the vectors stored in the database, we can efficiently query them using the `pgvecto.rs` extension. To find the nearest vectors to a given input text, we need to encode the text with the same model.

This process is identical to the one described in the previous chapter:

```python
def embedding(sentence: str) -> tuple[np.ndarray, list[list[int]], list[list[float]]]:
    """
    Create dense and sparse embeddings from text
    """
    output = model.encode(sentence, return_dense=True, return_sparse=True, return_colbert_vecs=False)
    dense_embedding = output['dense_vecs']
    sparse_weight_id = output['lexical_weights']
    sparse_indices = [int(token_id) for token_id in sparse_weight_id]
    sparse_values = [float(sparse_weight_id[token_id]) for token_id in sparse_weight_id]

    sparse_values = [x for _, x in sorted(zip(sparse_indices, sparse_values))]
    sparse_indices = sorted(sparse_indices)
    return dense_embedding, sparse_indices, sparse_values
```

The `pgvecto.rs` extension provides a simple `SELECT` statement to get sparse and dense vectors like this:

```python
dense_embedding, sparse_indices, sparse_values = embedding("The text you want to search...")

with psycopg.connect(URL) as conn:
    register_vector(conn)
    cur = conn.execute(
        "SELECT text FROM documents ORDER BY sparse <#> to_svector(%s, %s::real[]::int[], %s::real[]) LIMIT 50;",
        (len(model.tokenizer), sparse_indices, sparse_values),
    )
    sparse_result = cur.fetchall()
      
    cur = conn.execute(
              "SELECT text FROM documents ORDER BY dense <-> %s LIMIT 50;",
              (dense_embedding,),
          )
    dense_result = cur.fetchall()
    
# Merge sparse and dense vector search results
mix_text = set([r[0] for r in dense_result]).union([r[0] for r in sparse_result])
```

To rerank the merged result, you can use popular fusion methods such as Reciprocal Ranked Fusion (RRF). However, to make better use of `BGE-M3`, let's introduce the `Reranker` model of `BGE-M3`. Unlike the embedding model, `Reranker` uses text input and directly outputs their similarity or score.

We can initialize a `Reranker` model and feed it with mixed candidate texts. With this method, reranking can be as simple as sorting by the `Reranker` model's estimated score.

```python
from FlagEmbedding import FlagReranker

reranker = FlagReranker('BAAI/bge-reranker-large', use_fp16=True)

# Obtain relevance scores （Higher scores indicate greater relevance）
scores = reranker.compute_score([["The text you want to search...", candidate] for candidate in mix_text])
# Rerank text using combined dense and sparse embeddings
reranked_text = [t for _, t in sorted(zip(scores, mix_text))]
```

Congratulations! We have now completed our first step into the world of sparse vector search.