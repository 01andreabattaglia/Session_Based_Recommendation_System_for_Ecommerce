# Session-Based Recommendation System for E-commerce  
### A Comparative Study of Algorithmic Approaches to Product Matching

This project analyzes e-commerce session data to understand user behavior and build a session-based recommendation system for product pages.  
The work compares a **high-precision Cosine Similarity approach** with a **high-scalability SimHash + Locality-Sensitive Hashing (LSH) approach**, highlighting the trade-off between recommendation quality and computational efficiency. :contentReference[oaicite:0]{index=0}

---

## 🔍 Project Overview

Modern e-commerce platforms handle millions of products and users. Recommending relevant items requires methods that are:

- **Accurate** enough to capture nuanced user interests  
- **Scalable** enough to work on large, high-dimensional datasets  

In this project:

1. We start from **raw clickstream data** (views, add-to-cart, purchases).
2. We derive an **interest score** for each *(user, product)* pair based on:
   - Interaction type (view / cart / purchase)  
   - Dwell time on the product page  
3. We then compute **product–product similarities** using two strategies:
   - **Cosine Similarity** on normalized interest vectors  
   - **SimHash + LSH** on binary signatures with Hamming Similarity  

Finally, we **compare** both methods in terms of:
- Computational time
- Precision of recommended products vs. a brute-force Cosine Similarity baseline

---

## 🧾 Dataset

The dataset consists of **605,110 records** collected from an e-commerce website in an Asian country on **October 1, 2019 (00:00–12:00 UTC+0)**. Each row is an interaction between a user and a product page.

**Main fields:**

- `event_time` – timestamp of the interaction  
- `event_type` – `{view, cart, purchase}`  
- `product_id` – unique product identifier (~51K products)  
- `category_id`, `category_code` – product categories (~541 categories)  
- `brand` – product brand (~2K brands, many missing values)  
- `price` – product price (USD)  
- `user_id` – unique user (~106K users)  
- `user_session` – unique session (~142K sessions)

The analysis includes:
- Session-level behavior (pages per session / per user)
- Dwell time distribution and cleaning (removal of extreme values > 99th percentile)
- Time-zone interpretation of user activity

---

## ⭐ Interest Score Model

To transform raw interactions into a signal of **user interest**, we:

1. **Group repeated views** of the same product by the same user within a session.
2. Keep the most significant event according to:  
   `purchase > cart > view`.
3. Compute **dwell time** on a page as time between two consecutive events in the same session.
4. Model dwell times with a **lognormal distribution** (after log-transform, approximately normal).
5. Assign an **interest rating from 1 to 5** using quantiles:

- **1 – No Interest**  
  - View time below 0.01 quantile of view distribution  
- **2 – Neutral Interest**  
  - Between 0.01 and 0.05 quantile of view distribution  
- **3 – Moderate Interest**  
  - Between 0.05 quantile of view and 0.02 quantile of cart/purchase distribution  
- **4 – Strong Interest**  
  - Above 0.02 quantile of cart/purchase distribution, or last page of the session (dwell time unknown)  
- **5 – Definite Interest**  
  - Explicit `cart` or `purchase` event  

These scores become the base representation for user–product interactions.

<img width="1001" height="547" alt="image" src="https://github.com/user-attachments/assets/6df38999-e421-4188-94b5-55e814e8539e" />
<img width="1189" height="590" alt="image" src="https://github.com/user-attachments/assets/1a3e63ab-6bba-4153-a8c4-7e4445c14bf2" />


---

## 🧮 Methods

### 1. Category Similarity with Jaccard + MinHash

Before comparing products, we validate and exploit **category relationships**:

1. A category is considered *positively interacted with* in a session if at least one product in that category has **interest ≥ 3**.
2. For each pair of categories, we compute **Jaccard Similarity** over the set of sessions where each category is positively interacted with.
3. To scale this, we use **MinHash**:
   - Map sessions to integer IDs
   - Generate multiple random permutations (200 hash functions)
   - Build MinHash signatures for each category
   - Approximate Jaccard as the fraction of matching signature components  

This reveals **non-obvious but meaningful category overlaps**, such as:

- `computers.cpu` ↔ `computers.motherboard`  
- `apparel.jacket` ↔ `apparel.trousers`  
- `accessories.bag` ↔ `accessories.wallet`  

These similarity scores are later used to **limit product-pair comparisons** to:

- Same category  
- Pairs of categories with high Jaccard similarity

---

### 2. Cosine Similarity on Interest Vectors

We model each product as a vector of **centered interest scores** across users:

- First, we **center** the ratings by subtracting 2, so:
  - Values > 0 → positive interest  
  - Values ≤ 0 → low or no interest  

To avoid noise and unnecessary computation, we:

- Remove users who viewed **only one product**  
- Remove products viewed by **fewer than 3 users**

We then compute **Cosine Similarity**:

- Only **within the same category**
- Or **between categories** with high Jaccard similarity

This approach tends to deliver **high-quality recommendations**, but it is more expensive computationally on very large datasets.

---

### 3. SimHash + Locality-Sensitive Hashing (LSH)

To improve scalability, we implement **SimHash with random hyperplanes**:

1. Generate **200 random hyperplanes**, each defined by a random normal vector.
2. For each product vector:
   - Compute the dot product with each hyperplane  
   - If the result > 0 → bit = 1, else bit = 0  
3. Obtain a **200-bit binary signature** for each product.
4. Use **Hamming Similarity** (1 − normalized Hamming distance) between signatures as the similarity measure.

Again, we only compare:

- Products **within the same category**, and  
- Products in **similar categories** (from the Jaccard/MinHash step)

This dramatically reduces the cost of similarity computations, making the method **highly scalable**, at the price of some **loss in precision**.

---

## 📊 Results

### Computational Efficiency

For the *same number of comparisons* (~4.37M product pairs):

| Algorithm                                      | Total Comparisons | Total Time (s) |
|-----------------------------------------------|-------------------|----------------|
| Cosine Similarity (after category grouping)   | 4,371,878         | 102.17         |
| SimHash + LSH (random hyperplanes)           | 4,371,878         | 22.60          |

- SimHash is ~**5× faster** than Cosine Similarity for this workload.

### Recommendation Accuracy

Using brute-force Cosine Similarity as the **ground truth**:

- Both methods perform well when **true similarity > 0.4**.
- **Cosine Similarity**:
  - Higher precision across all similarity levels  
  - Retrieves meaningful relationships even at **moderate similarity**  
  - ~**80%** of recommended products appear in the top-20 most similar items
- **SimHash + LSH**:
  - Focuses mainly on the **strongest** similarities  
  - Tends to miss moderately similar items  
  - Precision around **45%** in the top-20 ranking
 
    <img width="1189" height="590" alt="image" src="https://github.com/user-attachments/assets/1954dd46-ce4e-44d4-aa24-33509218e154" />

    <img width="1189" height="590" alt="image" src="https://github.com/user-attachments/assets/d3646b22-02c2-4850-bcee-f6863b3dd2ad" />



**Trade-off:**

- Cosine Similarity → **best choice for quality** (when data size allows it)  
- SimHash + LSH → **best choice for scale** (huge datasets, many users / products)

A promising hybrid strategy is:

> Use **SimHash** to quickly filter candidate products,  
> then refine the **top candidates** with **Cosine Similarity**.
