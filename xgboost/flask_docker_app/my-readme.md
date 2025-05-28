# Examples from CHATGPT

Created these examples at the conclusion of [XGBoost for Regression | XGBoost Part 2 | CampusX](https://www.youtube.com/watch?v=gmp2tS2joaA)

---

### 🔄 Comparison of Project Archives

| Archive Name                             | Purpose                                                                                      | Frontend       | Backend Framework | ML Model       | Deployment        | MongoDB | Extras                                                         |
|-----------------------------------------|----------------------------------------------------------------------------------------------|----------------|--------------------|----------------|-------------------|---------|----------------------------------------------------------------|
| `xgboost_flask_app.zip`                 | Minimal Flask app for XGBoost regression                                                     | None           | Flask              | Manual or XGBoost | Flask only         | ❌      | Clean REST API to serve predictions                            |
| `xgboost_flask_app_docker.zip`          | Dockerized version of above for isolated, repeatable deployment                             | None           | Flask              | XGBoost         | Docker            | ❌      | `Dockerfile`, usage docs, REST endpoint for prediction         |
| `xgboost_flask_streamlit_mongodb.zip`   | Full-stack app with Streamlit UI, Flask API, and MongoDB vector store for persistence       | Streamlit      | Flask              | XGBoost         | Docker + Streamlit | ✅      | MongoDB container, REST + UI + SHAP + tuning + full notebook   |

---

### ✅ Key Differences

- **`xgboost_flask_app.zip`**:
  - Bare minimum for exposing an XGBoost model via Flask API.
  - Great for learning REST + XGBoost basics.
  - No containerization or frontend.

- **`xgboost_flask_app_docker.zip`**:
  - Adds Docker support for containerized deployment.
  - Still backend-only (API level).
  - Ideal for CI/CD and integration into backend workflows.

- **`xgboost_flask_streamlit_mongodb.zip`**:
  - Complete ML web app with frontend (Streamlit), API (Flask), persistence (MongoDB).
  - SHAP explanations and hyperparameter tuning included.
  - Docker Compose brings up MongoDB and app containers together.
  - Ready for production-like experimentation.

