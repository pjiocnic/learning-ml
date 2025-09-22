Awesome — here’s a clean, reproducible experiment plan you can run end-to-end to compare **TF-IDF vs. embeddings vs. hybrid** with **XGBoost**. It’s opinionated (sane defaults) but easy to tweak.

# 🧪 Experiment Plan: TF-IDF vs Embeddings vs Hybrid (XGBoost)

## 0) Goals & Design

* **Primary metric:** Macro-F1 (robust to class imbalance, good for multi-class).
* **Secondary:** Accuracy, ROC-AUC (binary/OVR), PR-AUC (if imbalanced), inference latency.
* **Fairness:** Same CV splits for all variants; stop when validation stops improving (early stopping).
* **Compare three setups:**

  1. TF-IDF → XGBoost
  2. Sentence embeddings (e.g., `all-MiniLM-L6-v2`) → XGBoost
  3. Hybrid (TF-IDF ⊕ Embeddings concatenated) → XGBoost (with dimensionality control)

---

## 1) Data Protocol

* **Split once** into **train/test** (e.g., 80/20 stratified).
* On **train**, run **Stratified K-Fold CV** (K=5). Use the **same folds** for all variants.
* Keep a **held-out test** untouched until final comparison.
* Text cleaning: minimal (lowercase, strip, optional basic URL/emoji removal). Avoid heavy normalization that may harm semantics.

```python
SEED = 42
TEST_SIZE = 0.2
N_SPLITS = 5
```

---

## 2) Baseline: TF-IDF → XGBoost

**Why:** fast, interpretable, strong baseline.

**Vectorizer parameters to tune:**

* `ngram_range`: (1,1) vs (1,2)
* `max_features`: \[20k, 50k]
* `min_df`: \[2, 5]
* `max_df`: \[0.9, 0.98]
* `sublinear_tf`: \[True]
* `norm`: \['l2']

**XGBoost knobs (start small):**

* `n_estimators`: early stopping with large cap (e.g., 2000)
* `learning_rate`: \[0.05, 0.1]
* `max_depth`: \[6, 8]
* `subsample`: \[0.8]
* `colsample_bytree`: \[0.7, 0.9]
* `reg_lambda`: \[1, 5]
* (binary only) `scale_pos_weight`: set to `neg/pos` ratio (or tune around it)

**Notes:**

* Use **sparse** input directly; XGBoost handles CSR well.
* Keep **feature count bounded** to avoid memory blowups.

---

## 3) Embeddings → XGBoost

**Why:** denser features, capture semantics & synonyms.

**Encoder:** `sentence-transformers/all-MiniLM-L6-v2` (384-dim, fast).
**Pooling:** default sentence embedding (already pooled).
**Cache** embeddings to disk (NPZ/Parquet) to avoid recompute.

**XGBoost knobs (typical):**

* `learning_rate`: \[0.05, 0.1]
* `max_depth`: \[8, 10, 12] (deeper is OK on low-dim dense features)
* `subsample`: \[0.8]
* `colsample_bytree`: \[0.8, 1.0]
* `reg_lambda`: \[1, 3]

**Variants to try later (optional):**

* Swap encoder: `all-mpnet-base-v2` (768-dim, stronger but slower).
* Add **dimensionality reduction** (e.g., PCA to 128) if you go to very large encoders.

---

## 4) Hybrid (TF-IDF ⊕ Embeddings) → XGBoost

**Why:** combine exact keyword signals (TF-IDF) with semantics (embeddings).

**How:** horizontally **stack** TF-IDF (sparse) with embeddings (dense).

* Make sure to **scale** the dense block (StandardScaler) if you add non-tree models; not strictly needed for XGBoost, but harmless.
* Keep TF-IDF vocabulary modest (e.g., ≤20k) to prevent feature explosion.
* If needed, apply **SVD** (TruncatedSVD) on TF-IDF to \~300–1000 dims before stacking.

---

## 5) Cross-Validation & Early Stopping

* Use **StratifiedKFold(n\_splits=5, shuffle=True, random\_state=SEED)**.
* Within each fold, split train→(train, valid) for early stopping (e.g., 90/10).
* Use `early_stopping_rounds=50` with a **large** `n_estimators` cap (e.g., 2000).
* Track **best\_iteration** and reuse it when scoring CV and test.

---

## 6) Evaluation Protocol

* **Per fold**: record Macro-F1, Accuracy, ROC-AUC (OVR if multiclass), confusion matrix.
* **Aggregate**: mean ± std across folds.
* **Threshold tuning** (binary or multilabel): optionally tune decision threshold using validation PR curve to optimize F1.
* **Calibration** (if you need well-calibrated probs): run `CalibratedClassifierCV` (sigmoid) on the trained model predictions.

---

## 7) Reproducible Skeleton (Python)

Below is a compact skeleton you can adapt. It avoids external logging frameworks; you can add MLflow later if desired.

```python
# pip install xgboost scikit-learn sentence-transformers scipy numpy pandas
import numpy as np, pandas as pd
from sklearn.model_selection import StratifiedKFold, train_test_split
from sklearn.metrics import f1_score, accuracy_score, roc_auc_score
from sklearn.feature_extraction.text import TfidfVectorizer
from xgboost import XGBClassifier
from sentence_transformers import SentenceTransformer
from scipy.sparse import csr_matrix, hstack

SEED = 42
TEST_SIZE = 0.2
N_SPLITS = 5

def evaluate_clf(clf, X, y):
    # Works for binary or multiclass; macro-F1 is primary
    y_pred = clf.predict(X)
    f1 = f1_score(y, y_pred, average='macro')
    acc = accuracy_score(y, y_pred)
    # ROC-AUC robust handling (skip if not supported)
    roc = np.nan
    try:
        proba = clf.predict_proba(X)
        if len(np.unique(y)) == 2:
            roc = roc_auc_score(y, proba[:,1])
        else:
            roc = roc_auc_score(y, proba, multi_class='ovr')
    except Exception:
        pass
    return {'macro_f1': f1, 'accuracy': acc, 'roc_auc': roc}

def cv_run(X, y, build_model_fn, n_splits=5, seed=SEED):
    skf = StratifiedKFold(n_splits=n_splits, shuffle=True, random_state=seed)
    metrics = []
    for tr_idx, va_idx in skf.split(X, y):
        X_tr, X_va = X[tr_idx], X[va_idx]
        y_tr, y_va = y[tr_idx], y[va_idx]

        clf = build_model_fn()
        # XGBoost early stopping: pass eval_set
        clf.fit(X_tr, y_tr,
                eval_set=[(X_va, y_va)],
                eval_metric='mlogloss' if len(np.unique(y)) > 2 else 'logloss',
                verbose=False,
                early_stopping_rounds=50)
        # Refit on (X_tr + X_va) at best_iteration if you want; or just evaluate now:
        fold_metrics = evaluate_clf(clf, X_va, y_va)
        metrics.append(fold_metrics)
    df = pd.DataFrame(metrics)
    return df.mean().to_dict(), df.std().to_dict()

# -----------------------
# Example dataset schema
# texts: list/Series[str]
# labels: np.array[int] or str (encode to int first)
# -----------------------
# 1) Train/test split
# texts, labels = ...
# encode labels to int if needed
# from sklearn.preprocessing import LabelEncoder
# le = LabelEncoder(); y = le.fit_transform(labels)
# X_train_txt, X_test_txt, y_train, y_test = train_test_split(
#     texts, y, test_size=TEST_SIZE, random_state=SEED, stratify=y
# )

# ---------- Variant A: TF-IDF ----------
def build_tfidf_features(train_texts, test_texts):
    tfidf = TfidfVectorizer(
        ngram_range=(1,2),
        max_features=30000,
        min_df=5,
        max_df=0.9,
        sublinear_tf=True,
        norm='l2'
    )
    X_tr = tfidf.fit_transform(train_texts)
    X_te = tfidf.transform(test_texts)
    return X_tr, X_te, tfidf

def build_xgb(num_classes):
    params = dict(
        n_estimators=2000,
        learning_rate=0.1,
        max_depth=7,
        subsample=0.8,
        colsample_bytree=0.8,
        reg_lambda=3.0,
        random_state=SEED,
        tree_method='hist'   # 'gpu_hist' if you have GPU
    )
    if num_classes == 2:
        clf = XGBClassifier(objective='binary:logistic', **params)
    else:
        clf = XGBClassifier(objective='multi:softprob', num_class=num_classes, **params)
    return clf

# ---------- Variant B: Embeddings ----------
def embed_texts(model_name, texts, batch_size=64):
    model = SentenceTransformer(model_name)
    # returns float32 dense array
    X = model.encode(texts, batch_size=batch_size, show_progress_bar=False, convert_to_numpy=True)
    return X

# ---------- Variant C: Hybrid ----------
def hstack_sparse_dense(X_sparse, X_dense):
    if not isinstance(X_sparse, csr_matrix):
        X_sparse = csr_matrix(X_sparse)
    X_dense_csr = csr_matrix(X_dense)
    return hstack([X_sparse, X_dense_csr]).tocsr()

# ====== Run the three experiments (sketch) ======
# X_train_tfidf, X_test_tfidf, _ = build_tfidf_features(X_train_txt, X_test_txt)
# num_classes = len(np.unique(y_train))

# A) TF-IDF
# def build_model_A():
#     return build_xgb(num_classes)
# meanA, stdA = cv_run(X_train_tfidf, y_train, build_model_A, n_splits=N_SPLITS)
# clfA = build_model_A()
# clfA.fit(X_train_tfidf, y_train, eval_set=[(X_test_tfidf, y_test)],
#          eval_metric='mlogloss' if num_classes > 2 else 'logloss',
#          early_stopping_rounds=50, verbose=False)
# testA = evaluate_clf(clfA, X_test_tfidf, y_test)

# B) Embeddings
# X_train_emb = embed_texts('all-MiniLM-L6-v2', list(X_train_txt))
# X_test_emb  = embed_texts('all-MiniLM-L6-v2', list(X_test_txt))
# def build_model_B():
#     return build_xgb(num_classes)
# meanB, stdB = cv_run(X_train_emb, y_train, build_model_B, n_splits=N_SPLITS)
# clfB = build_model_B()
# clfB.fit(X_train_emb, y_train, eval_set=[(X_test_emb, y_test)],
#          eval_metric='mlogloss' if num_classes > 2 else 'logloss',
#          early_stopping_rounds=50, verbose=False)
# testB = evaluate_clf(clfB, X_test_emb, y_test)

# C) Hybrid
# X_train_h = hstack_sparse_dense(X_train_tfidf, X_train_emb)
# X_test_h  = hstack_sparse_dense(X_test_tfidf, X_test_emb)
# def build_model_C():
#     return build_xgb(num_classes)
# meanC, stdC = cv_run(X_train_h, y_train, build_model_C, n_splits=N_SPLITS)
# clfC = build_model_C()
# clfC.fit(X_train_h, y_train, eval_set=[(X_test_h, y_test)],
#          eval_metric='mlogloss' if num_classes > 2 else 'logloss',
#          early_stopping_rounds=50, verbose=False)
# testC = evaluate_clf(clfC, X_test_h, y_test)

# # Summary table example:
# # results = pd.DataFrame({
# #   'variant': ['TFIDF', 'EMB', 'HYBRID'],
# #   'cv_macro_f1_mean': [meanA['macro_f1'], meanB['macro_f1'], meanC['macro_f1']],
# #   'cv_macro_f1_std':  [stdA['macro_f1'],  stdB['macro_f1'],  stdC['macro_f1']],
# #   'test_macro_f1':    [testA['macro_f1'], testB['macro_f1'], testC['macro_f1']],
# #   'test_accuracy':    [testA['accuracy'], testB['accuracy'], testC['accuracy']],
# #   'test_roc_auc':     [testA['roc_auc'],  testB['roc_auc'],  testC['roc_auc']],
# # })
# # print(results.sort_values('test_macro_f1', ascending=False))
```

---

## 8) Hyperparameter Tuning Strategy (fast → thorough)

1. **Coarse search** with small grids (above).
2. Lock best **vectorizer/encoder** choice; then **refine XGBoost** (depth, learning\_rate, reg\_lambda).
3. Add **class weighting** / `scale_pos_weight` if imbalanced.
4. For hybrid: optionally **TruncatedSVD(300–500)** on TF-IDF before stacking to keep total dims reasonable.

---

## 9) Reporting Checklist (per variant)

* CV scores (mean ± std): Macro-F1, Accuracy, ROC-AUC.
* Test scores on held-out.
* Confusion matrix (per-class precision/recall).
* **Latency**: avg embedding time / sample (for embeddings), model inference time / sample.
* **Interpretability**:

  * TF-IDF: top positive/negative features by gain/cover.
  * Embeddings: nearest-neighbor probes for representative samples.
* **Size**: feature dimensionality, model size (MB).

---

## 10) Pitfalls & Guardrails

* **Data leakage:** Do not vectorize on full corpus. Fit TF-IDF only on train (per fold).
* **Recompute cost:** Cache embeddings to disk.
* **Memory blowups:** Cap TF-IDF `max_features`; consider SVD for hybrid.
* **Thresholds:** If classes are skewed, tune classification threshold on validation PR curve.

---

## 11) Optional: Extras for Production-ish Runs

* **GPU**: set `tree_method='gpu_hist'` if available.
* **MLflow**: log params/metrics/artifacts per variant.
* **Calibration**: wrap best model in `CalibratedClassifierCV` if you need reliable probabilities.
* **Ensembles**: soft-vote TF-IDF-XGB and Emb-XGB; sometimes beats hybrid concatenation.

---

If you want, I can turn this into a **ready-to-run notebook** with a small public dataset (e.g., 20 Newsgroups) and produce a results table comparing the three variants. Or I can adapt it to your own dataset schema (column names, label encoding, etc.) and include MLflow logging + SHAP/feature importance views.
