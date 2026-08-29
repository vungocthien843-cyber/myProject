# Improving Drug–Drug Interaction Prediction

Dự án dự đoán tương tác thuốc–thuốc (Drug–Drug Interaction, DDI) từ DrugBank 5.0. Notebook chính: [`processData_Drugbank5.0_Final_.ipynb`](processData_Drugbank5.0_Final_.ipynb).

Pipeline kết hợp đặc trưng cấu trúc hóa học, ngữ nghĩa y sinh và cấu trúc mạng tương tác thuốc; sau đó đánh giá bằng nhiều mô hình ML, SHAP/ablation và ensemble.

## 1. Luồng xử lý



```text
DrugBank XML → làm sạch nodes/edges → ghép SIDER
→ similarity ATC/MeSH/ADE/SMILES → đồ thị DDI + Louvain
→ 8 topology features → ghép thành 12 features
→ train/test chống leakage → benchmark, SHAP/ablation, ensemble
```

## 2. Các bước chính

### 2.1. Trích xuất và làm sạch DrugBank

`DrugBankParser` đọc XML và tạo `drug_nodes.csv`, `drug_edges.csv`. Một thuốc chỉ được giữ khi có SMILES, InChIKey, ATC và MeSH; cạnh trỏ tới thuốc không hợp lệ bị loại.

`DDI_NetworkFilter` lọc cạnh theo node hợp lệ, chuẩn hóa cạnh vô hướng (`source < target`), loại cạnh trùng/self-loop và thuốc cô lập. Đầu ra: `drug_node_ver2.csv`, `drug_edge_final.csv`.

### 2.2. Ghép tác dụng phụ từ SIDER

`SIDER_DrugMapper` ánh xạ DrugBank → PubChem CID, ưu tiên InChIKey và dự phòng DrugBank ID qua PubChem PUG REST API. Các mã UMLS/MedDRA được gom vào `side_effect_codes`.

Đầu ra: `drug_nodes_with_side_effects.csv` và `drug_node_final.csv` (bỏ `inchikey`, `pubchem_cid`).

### 2.3. Tạo đặc trưng semantic

ATC, MeSH và ADE được chuyển thành vector nhị phân, TF-IDF, chuẩn hóa L2 rồi tính cosine similarity. SMILES được chuyển thành Morgan fingerprint (radius 2, 1024 bit) và tính Tanimoto similarity.

Đầu ra:

```text
feature_ATC.csv, feature_MESH.csv, feature_ADE.csv, feature_SMILES.csv
```

Bốn giá trị được ghép theo cặp thuốc thành `sim_atc`, `sim_mesh`, `sim_ade`, `sim_chemical`.

### 2.4. Xây dựng đồ thị và Louvain

`DDIGraphBuilder` tạo đồ thị vô hướng NetworkX từ `drug_edge_final.csv`, gắn thuộc tính node và lưu `ddi_graph_final.pkl`.

`SimpleLouvainPipeline` chia cạnh dương thành train/test với `test_size=0.34`. `G_train` chứa toàn bộ node nhưng chỉ chứa cạnh dương train; Louvain được chạy trên `G_train` để không dùng cạnh test. Kết quả: `drug_communities.csv`, `ddi_graph_train_with_comm.pkl`, `pos_edge_split.pkl`.

### 2.5. Tạo 8 đặc trưng topology

Tất cả cặp thuốc được tính trên `G_train`:

| Nhóm           | Đặc trưng                             |
| --------------- | ---------------------------------------- |
| Neighborhood    | `cn`, `jc`, `aai`, `rai`, `pa` |
| Community-aware | `ccn`, `cra`, `wic`                |

`label=1` nếu cặp là cạnh DrugBank, `label=0` nếu không có cạnh. `pos_split` ghi cạnh dương thuộc train/test. Đầu ra: `8ft_topology.csv`.

### 2.6. Ghép dataset và chia train/test

`DataIntegrationPipeline` ghép 8 topology + 4 semantic thành `Final_Dataset_DDI.csv`.

`build_final_train_test` giữ nguyên `pos_split` của cạnh dương, chỉ chia mẫu âm ngẫu nhiên theo tỷ lệ test 34%, với `random_state=42`. Sau đó bỏ các cột hỗ trợ `u`, `v`, `pos_split` và lưu:

```text
train_data.csv
test_data.csv
```

## 3. Đánh giá mô hình

Benchmark gồm Decision Tree, Logistic Regression, Gaussian Naive Bayes, kNN, SVM, Random Forest, XGBoost và LightGBM. Các chỉ số gồm F1, AUPR, Precision, Recall, Accuracy, Specificity, Log loss và thời gian chạy. Do dữ liệu mất cân bằng, cần ưu tiên F1/AUPR thay vì chỉ Accuracy.

Phần phân tích đặc trưng gồm:

1. Leave-one-out: bỏ từng feature và đo thay đổi F1/AUPR.
2. SHAP cumulative: xếp hạng theo `|SHAP|` trên train rồi thêm dần feature.
3. Group ablation: so sánh toàn bộ, chỉ topology, chỉ semantic, bỏ ADE/ATC.

Ensemble so sánh RF, SVM, GaussianNB, Logistic Regression với Hard Voting, Soft Voting và Stacking (`cv=5`, `passthrough=True`). cuML được dùng khi có GPU; nếu không sẽ fallback sang scikit-learn CPU.

## 4. Cách chạy

Notebook hiện dành cho Kaggle, sử dụng `/kaggle/input` và `/kaggle/working`. Cần cung cấp:

- DrugBank XML, hiện dùng tên `full database.xml`;
- SIDER `meddra_all_se.tsv`;
- `train_data.csv` và `test_data.csv` nếu chạy trực tiếp phần đánh giá.

Chạy các cell theo đúng thứ tự vì mỗi bước dùng file sinh ra từ bước trước. Có thể cần:

```bash
pip install rdkit python-louvain shap xgboost lightgbm
```

Khi chạy local, phải thay các đường dẫn Kaggle. GPU/cuML là tùy chọn; CPU vẫn chạy được nhưng có thể rất chậm.

## 5. Điều cần chú ý

1. **Không commit DrugBank XML.** Dữ liệu DrugBank có bản quyền/hạn chế sử dụng; file lớn đã được khai báo trong `.gitignore`.
2. **Chống leakage:** community và topology phải học từ `G_train`, không dùng cạnh test.
3. **Không chia lại cạnh dương:** luôn dùng `pos_edge_split.pkl` và `pos_split` ở bước cuối.
4. **Không chọn feature/model bằng test nhiều lần:** chọn bằng CV trên train; test chỉ xác nhận cuối.
5. **Tốn bộ nhớ:** notebook tạo ma trận similarity `N×N` và mọi cặp thuốc, nên có thể cần giảm dữ liệu, dùng sparse/batch hoặc máy nhiều RAM.
6. **PubChem cần Internet:** API có thể rate-limit hoặc không tìm thấy CID. Nên lưu CSV trung gian để tránh gọi lại.
7. **Giữ thứ tự `drugbank_id`:** thứ tự hàng/cột của bốn ma trận similarity phải nhất quán khi lookup.
8. **Tái lập kết quả:** giữ `random_state=42`, ghi lại seed, phiên bản thư viện và chế độ GPU/CPU.

## 6. File trung gian

| File                                                                                   | Vai trò                             |
| -------------------------------------------------------------------------------------- | ------------------------------------ |
| `drug_nodes.csv`, `drug_edges.csv`                                                 | Node/cạnh trích xuất từ DrugBank |
| `drug_node_ver2.csv`, `drug_edge_final.csv`                                        | Mạng sau làm sạch                 |
| `drug_nodes_with_side_effects.csv`, `drug_node_final.csv`                          | Node đã ghép SIDER                |
| `feature_ATC.csv`, `feature_MESH.csv`, `feature_ADE.csv`, `feature_SMILES.csv` | Ma trận tương đồng              |
| `ddi_graph_final.pkl`                                                                | Đồ thị DDI đầy đủ             |
| `ddi_graph_train_with_comm.pkl`                                                      | Đồ thị train có community        |
| `pos_edge_split.pkl`                                                                 | Cạnh dương train/test             |
| `8ft_topology.csv`                                                                   | 8 topology features và nhãn        |
| `Final_Dataset_DDI.csv`                                                              | Dataset đủ 12 features             |
| `train_data.csv`, `test_data.csv`                                                  | Dữ liệu đầu vào mô hình       |
