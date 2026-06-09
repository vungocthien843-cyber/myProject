Data Source: DrugBank 5.0
## Data Processing Pipeline

The project adopts a **Data-Centric** strategy with a rigorous curation process to preserve the realistic distribution of biomedical data, rather than utilizing artificial balancing techniques. The pipeline consists of the following steps:

* **Step 1: Data Curation**
  * Raw data is extracted from the DrugBank database.
  * The negative samples are kept intact to maintain the natural imbalanced distribution, which accurately reflects the actual rate of Drug-Drug Interactions (DDI) in real-world pharmacological environments.

* **Step 2: Multi-View Feature Extraction**
  * The original feature space is constructed with **12 dimensions**, combining:
    * *Topological Features*: Extracted from the drug interaction network.
    * *Semantic Features*: Relevant biochemical attributes.

* **Step 3: Feature Evaluation & Refinement**
  * The **SHAP (SHapley Additive exPlanations)** tool is applied to the Random Forest model to quantify the contribution level of each feature.
  * An ablation study is conducted to identify and eliminate noisy features (such as ATC and ADE). 
  * *Result:* The data space is optimized from the initial 12 features down to **10 core features**.

* **Step 4: Data Splitting & Cross-Validation**
  * The data is standardized and partitioned using a **5-Fold Cross-Validation** strategy to ensure objectivity and control overfitting during the training of the classification models and the Stacking Ensemble architecture.
