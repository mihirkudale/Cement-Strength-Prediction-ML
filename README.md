# Cement Strength Prediction ML

End‑to‑end machine learning pipeline to predict the **compressive** strength of concrete/cement based on its mix design and curing time.  
This project covers automated data validation, database‑backed ingestion, preprocessing, model training with model selection, and a Flask web app for batch predictions.

---

## 1. Problem overview

Concrete mix design directly controls structural safety, durability, and cost.  
The goal of this project is to build a regression model that predicts compressive strength (target) from raw mix data so that civil engineers can virtually test different mix designs before casting.

Typical input features include:

- Cement content  
- Blast furnace slag, fly ash, water  
- Superplasticizer, coarse aggregate, fine aggregate  
- Age (curing time in days)

Output:

- Compressive strength (e.g., MPa) as a continuous value.

---

## 2. Architecture and flow

At a high level, the system is designed as a production‑like ML pipeline with clearly separated training and prediction flows and strong input governance.

### 2.1 High‑level workflow

1. Raw batch files (CSV) are dropped into dedicated training or prediction batch folders.  
2. Raw file structure is validated against JSON schemas, and invalid files are archived with logs.  
3. Valid data is loaded into a relational database (SQLite) for traceability and consistency.  
4. Data is exported from DB, preprocessed (missing values, scaling, etc.), and split for model training.  
5. Multiple algorithms are trained and tuned; the best model is selected and serialized.  
6. For prediction, the same schema validation and preprocessing steps are applied before using the saved model.  
7. Predictions are written to an output file and surfaced via a Flask web UI.

### 2.2 Tech stack

- Python, scikit‑learn, pandas, NumPy  
- Flask for the web application and monitoring integration  
- SQLite for training and prediction databases  
- Custom logging utilities for end‑to‑end observability[page:2]

---

## 3. Repository structure

Core directories and files:

- `Training_Batch_Files/` – Raw CSVs used to train the model.  
- `Training_Raw_data_validation/` – Validates structure, file names, and column types of training files.  
- `DataTypeValidation_Insertion_Training/` – Validates data types and inserts validated training data into the training database.  
- `Training_Database/`, `Training_FileFromDB/` – SQLite DB and exported training data files.  
- `DataTransform_Training/` – Training‑time data transformation utilities (e.g., handling categorical/numerical features).  
- `Training_Logs/`, `TrainingArchiveBadData/` – Logs for training pipeline steps and archived invalid training batches.

Prediction side mirrors the above:

- `Prediction_Batch_files/` – Raw CSVs for prediction.  
- `Prediction_Raw_Data_Validation/` – Structural and schema validation for prediction files.  
- `DataTypeValidation_Insertion_Prediction/` – Data type checks and insertion into prediction database.  
- `Prediction_Database/`, `Prediction_FileFromDB/` – SQLite DB and exported prediction data files.  
- `DataTransformation_Prediction/` – Prediction‑time data transformation consistent with training.  
- `Prediction_Logs/`, `PredictionArchivedBadData/` – Logs and archived invalid prediction files.  
- `Prediction_Output_File/` – Final prediction CSVs consumed by the app or user.

Shared/utility modules:

- `data_ingestion/` – High‑level ingestion components used by both training and prediction pipelines.  
- `data_preprocessing/` & `preprocessing_data/` – Preprocessing utilities (missing value handling, scaling, encoding).  
- `best_model_finder/` – Hyperparameter tuning and algorithm comparison to pick the best regression model.  
- `file_operations/` – Model save/load operations (e.g., pickle).  
- `models/` – Directory containing persisted model artifacts.  
- `application_logging/` – Custom logging wrapper used across the project.  
- `templates/` – HTML templates for the Flask web interface.  
- `EDA/` – Jupyter notebooks for exploratory data analysis.

Top‑level files:

- `trainingModel.py` – Orchestrates the full training pipeline.  
- `training_Validation_Insertion.py` – Entry for training data validation and insertion.  
- `predictFromModel.py` – Orchestrates the full prediction pipeline.  
- `prediction_Validation_Insertion.py` – Entry for prediction data validation and insertion.  
- `main.py` – Flask application exposing web UI for training and prediction.  
- `schema_training.json`, `schema_prediction.json` – Column names, lengths, and data type expectations for validation.  
- `requirements.txt` – Python dependencies.  
- `app.yaml`, `manifest.yml` – Configuration for deployment on cloud platforms.

---

## 4. End‑to‑end training pipeline

The training pipeline is implemented to be repeatable, auditable, and close to MLOps best practices for tabular regression.

### 4.1 Input schema and raw validation

- Training CSVs placed in `Training_Batch_Files/` must match `schema_training.json` in terms of file name pattern, number of columns, column names and order, data types and allowable lengths.  
- `Training_Raw_data_validation` reads the schema, validates each incoming file, logs all operations in `Training_Logs/`, and moves invalid files to `TrainingArchiveBadData/`.

### 4.2 Database insertion

- Validated records are inserted into a training SQLite database in `Training_Database/` via the `DataTypeValidation_Insertion_Training` and `data_ingestion` modules.  
- A clean training dataset is then exported as CSV into `Training_FileFromDB/` to decouple downstream stages from direct DB access.

### 4.3 Data transformation and preprocessing

- `DataTransform_Training` and `data_preprocessing` handle cleaning and feature engineering.  
- Steps include handling missing values, outlier treatment, scaling/normalization of numerical features, and any required encoding.  
- The preprocessing objects (e.g., scalers, encoders) are saved so that the same transformations can be applied during prediction.

### 4.4 Model training and selection

- `trainingModel.py` loads the transformed data, splits into train and test, and invokes `best_model_finder`.  
- `best_model_finder` trains multiple regression algorithms and performs hyperparameter tuning using validation metrics.  
- The best performing model (e.g., highest \(R^2\), lowest RMSE) is serialized via `file_operations` into `models/`.

### 4.5 Logging and monitoring

- Every step—validation, DB operations, transformations, training, and model persistence—is logged with timestamps using `application_logging` into `Training_Logs/`.  
- `flask_monitoringdashboard.db` allows monitoring of API endpoints once the Flask app is running.

---

## 5. End‑to‑end prediction pipeline

The prediction pipeline mirrors training to guarantee that production inputs are treated identically to training data.

### 5.1 Batch input and validation

- New data for prediction is placed in `Prediction_Batch_files/`.  
- `prediction_Validation_Insertion.py` reads `schema_prediction.json` and validates each file for correct file name pattern, column count and order, data types and lengths.  
- Invalid files are moved to `PredictionArchivedBadData/` and all issues are logged in `Prediction_Logs/`.

### 5.2 DB insertion and preprocessing

- Validated prediction records are inserted into `Prediction_Database/` using `DataTypeValidation_Insertion_Prediction`.  
- Clean data is exported into `Prediction_FileFromDB/` and passed to `DataTransformation_Prediction` and `data_preprocessing` to apply the same transformations used in training.

### 5.3 Inference and outputs

- `predictFromModel.py` loads the latest saved model from `models/` using `file_operations`.  
- Predictions are generated for the transformed input data and written to CSV files in `Prediction_Output_File/`.  
- These outputs can be downloaded via the Flask UI or consumed by downstream systems.

---

## 6. Flask web application

The project ships with a Flask app (`main.py`) and HTML templates for an engineer‑friendly interface.

Key capabilities:

- Web page (from `templates/`) to upload training or prediction batch files.  
- Buttons/endpoints to trigger training or prediction pipelines end‑to‑end.  
- Display and download of prediction outputs (final CSV with compressive strength per row).  
- Integrated performance monitoring via `flask_monitoringdashboard.db`.

---

## 7. Setup and installation

1. Clone the repository:

   ```bash
   git clone https://github.com/mihirkudale/Cement-Strength-Prediction-ML.git
   cd Cement-Strength-Prediction-ML
   ```

2. Create and activate a virtual environment:

   ```bash
   python -m venv venv
   source venv/bin/activate   # On Windows: venv\Scripts\activate
   ```

3. Install dependencies:

   ```bash
   pip install -r requirements.txt
   ```

All required libraries for Flask, data processing, and modeling are declared in `requirements.txt`.

---

## 8. How to run

### 8.1 Run end‑to‑end training (CLI)

1. Place your training CSV files in `Training_Batch_Files/` following the format in `schema_training.json`.  
2. From the project root, run:

   ```bash
   python trainingModel.py
   ```

3. Check `Training_Logs/` for detailed logs and `models/` to confirm that the trained model artifacts are created.

### 8.2 Run batch prediction (CLI)

1. Place your prediction CSV files in `Prediction_Batch_files/` following `schema_prediction.json`.  
2. From the project root, run:

   ```bash
   python predictFromModel.py
   ```

3. Collect results from `Prediction_Output_File/` and review `Prediction_Logs` for any validation or processing messages.

### 8.3 Run the Flask web app

1. Ensure dependencies are installed and the environment is activated.  
2. Start the app:

   ```bash
   python main.py
   ```

3. Open the local URL printed in the terminal (typically `http://127.0.0.1:5000/`) in a browser and use the UI to upload files and trigger training/prediction.

---

## 9. Deployment notes

The repository includes configuration files for cloud deployment:

- `app.yaml` – App configuration suitable for platforms like Google App Engine.  
- `manifest.yml` – Manifest file compatible with Cloud Foundry‑style environments.

A standard deployment flow would be:

- Package the application including `main.py`, templates, and model artifacts.  
- Configure environment variables, logging, and storage paths for batch files.  
- Deploy using the platform‑specific CLI and point traffic to the Flask app entrypoint.

---

## 10. Logging, monitoring, and observability

- All pipelines use `application_logging` to create step‑wise log files under `Training_Logs` and `Prediction_Logs`.  
- Logs include success/failure of validation, DB operations, preprocessing, model training, and inference.  
- `flask_monitoringdashboard.db` integrates with the Flask app to monitor request‑level metrics in production environments.

---

## 11. Future improvements

Some natural extensions to this project:

- Add CI/CD with GitHub Actions for automated tests and linting on push.  
- Containerize the app using Docker and deploy on Kubernetes for scalable inference.  
- Integrate model versioning and experiment tracking tools (e.g., MLflow) for more robust governance.  
- Expose a REST API endpoint for real‑time scoring in addition to batch prediction.

---

## 12. License

This project is licensed under the MIT License as defined in the `LICENSE` file.
