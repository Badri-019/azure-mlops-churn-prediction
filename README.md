# 📊 Customer Churn Prediction

A machine learning project to predict customer churn using behavioral and billing data.

---

## 📌 Problem Statement

Predict whether a customer will leave (churn) based on features like tenure, charges, and support interactions.

---

## 🧠 Model Overview

- Algorithm: Scikit-learn (classification)
- Input Features:
  - Age
  - Tenure (months)
  - Monthly Charges
  - Total Charges
  - Number of Support Calls

### Example Prediction

```json
{
  "churn": 1,
  "churn_probability": 0.73
}

📂 Project Structure

azure-mlops/
│── train.py
│── generate_data.py
│── api.py
│── requirements.txt
│── models/
│── data/

🚀 Local Setup

pip install -r requirements.txt
python generate_data.py
python train.py
python api.py

🌐 API Usage

curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "age": 45,
    "tenure_months": 24,
    "monthly_charges": 79.99,
    "total_charges": 1920.00,
    "num_support_calls": 3
  }'

🎯 Key Highlights

Built a machine learning model for churn prediction
Exposed model via FastAPI
Structured project for scalability and future MLOps integration

🚧 Next Steps

Add model versioning using DVC
Deploy using Kubernetes (AKS)
Implement CI/CD pipeline
