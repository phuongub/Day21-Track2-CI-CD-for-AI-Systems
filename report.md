# Bao Cao Lab MLOps — Day 21 CI/CD cho AI Systems

## 1. Bo sieu tham so duoc chon

- **n_estimators:** 100
- **max_depth:** 20
- **min_samples_split:** 2

**Ly do:** Chay thu nghiem cuc bo voi 9+ cau hinh khac nhau (50-1000 cay, do sau 3-None). Cau hinh tren dat accuracy cao nhat tren tap eval: **0.6840**. Cac gia tri lon hon (200, 300, 500 cay) khong cai thien dang ke, cho thay du lieu Wine Quality co tinh noisy va RandomForest da bi overfit/ plateau.

## 2. Kho khan va cach giai quyet

| Kho khan | Cach giai quyet |
|---|---|
| **AWS credentials trong GitHub Actions** — DVC loi "Unable to locate credentials" nhieu lan do bien moi truong rong | Xac dinh nguyen nhan la JSON secret bi malformated (co khoang trang hoac xuong dong). Xoa va tao lai secret `CLOUD_CREDENTIALS` voi dung dinh dang 1-dong compact JSON: `{"aws_access_key_id":"...","aws_secret_access_key":"..."}`. Pipeline chay ngay lan sau. |
| **PowerShell escaping khi curl /predict** | PowerShell bien doi dau nhay kep trong JSON. Giai quyet bang cach ghi payload ra file tam roi dung `curl.exe -d "@file.json"`. |
| **Eval gate 0.70 ban dau khong qua** (acc 0.684) | Day la hanh vi dung. O Buoc 3, sau khi bo sung 2998 mau du lieu moi, accuracy vuot nguong 0.70 va pipeline chay xanh toan bo — chung minh quy trinh tu dong hoa hoat dong. |

## 3. Ket qua tong quan

- Buoc 1: MLflow UI ghi nhan 24 lan chay thi nghiem.
- Buoc 2: CI/CD hoan chinh 4 jobs (Test → Train → Eval → Deploy).
- Buoc 3: Push du lieu moi kich hoat pipeline tu dong, 4 jobs deu xanh, model duoc trien khai lai tren VM.
- Endpoint `/health` va `/predict` hoat dong chinh xac tren EC2.
