# Project Plan: VN30F1M Intraday Trading AI Model

> **Status:** 📋 PLANNING  
> **Last Updated:** 2026-01-03

## 1. Requirement

Xây dựng một application có các chức năng chính:

1. ✅ **DONE** - Dựa vào dữ liệu đã thu thập để build ra các features → `VN30FeaturePipeline`
2. Dùng AI sử dụng features này để dự đoán giá trong phiên của VN30 (hợp đồng phái sinh VN30F1M). Lưu ý là chỉ nắm giữ trong phiên, bắt buộc sẽ bán khi đặt target profit, chạm stop loss hoặc cuối phiên.
   
Cụ thể AI model cần sẽ đánh giá, tức là sẽ mở lệnh và đóng lệnh tại các thời điểm phù hợp trong ngày, đưa ra hành động với 3 trường hợp:

- Vào vị thế và dự đoán Giá tăng X%
- Vào vị thế và dự đoán Giá giảm X%
- Không vào vị thế

---

## 2. AI Models cho Trading Prediction - Giải thích chi tiết

### 2.1 Tổng quan các loại AI Models

Trong thực tế, với bài toán dự đoán giá phái sinh như của bạn, có **4 nhóm chính** của AI models:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AI MODELS CHO TRADING PREDICTION                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │
│  │ 1. TREE-BASED   │  │ 2. DEEP LEARNING│  │ 3. REINFORCEMENT│          │
│  │    MODELS       │  │    (Neural Net) │  │    LEARNING     │          │
│  │                 │  │                 │  │                 │          │
│  │ • Random Forest │  │ • LSTM          │  │ • DQN           │          │
│  │ • XGBoost       │  │ • GRU           │  │ • PPO           │          │
│  │ • LightGBM ⭐   │  │ • Transformer   │  │ • A2C           │          │
│  │ • CatBoost      │  │ • TCN           │  │                 │          │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘          │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ 4. ENSEMBLE & HYBRID                                        │        │
│  │                                                             │        │
│  │ Kết hợp nhiều models để tăng độ chính xác                   │        │
│  │ • Stacking: LightGBM + LSTM                                 │        │
│  │ • Voting: Nhiều models vote cho quyết định                  │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.2 Nhóm 1: Tree-Based Models (Gradient Boosting)

#### 🌲 Cách hoạt động (Giải thích đơn giản)

Hãy tưởng tượng bạn hỏi 100 chuyên gia khác nhau: "Giá sẽ tăng hay giảm?"

- Mỗi chuyên gia là một "cây quyết định" (decision tree)
- Mỗi cây nhìn vào các features (shark buy value, volume...) và đưa ra dự đoán
- Kết quả cuối cùng = trung bình của tất cả các cây

**Gradient Boosting** cải tiến thêm:
- Cây sau học từ sai lầm của cây trước
- Tập trung vào những trường hợp khó dự đoán

#### 📊 So sánh các Tree-Based Models

| Model | Tốc độ | Độ chính xác | Đặc điểm nổi bật |
|-------|--------|--------------|------------------|
| **Random Forest** | Nhanh | Tốt | Đơn giản, ít overfit |
| **XGBoost** | Trung bình | Rất tốt | Regularization mạnh |
| **LightGBM** ⭐ | Rất nhanh | Rất tốt | Tốt nhất cho tabular data |
| **CatBoost** | Chậm | Rất tốt | Tốt với categorical features |

#### ✅ Ưu điểm

```
✓ Cực kỳ phù hợp với tabular/structured data (như features của bạn)
✓ Tốc độ train và inference nhanh
✓ Không cần nhiều data preprocessing (normalization, etc.)
✓ Feature importance rõ ràng → biết feature nào quan trọng
✓ Ít bị overfit nếu tune đúng cách
✓ Dễ debug và interpret kết quả
```

#### ❌ Nhược điểm

```
✗ Không capture được temporal patterns tốt (chuỗi thời gian)
✗ Mỗi sample được xử lý độc lập (không nhớ context trước đó)
✗ Cần feature engineering thủ công (lag features, rolling features)
```

#### 🎯 Khi nào nên dùng?

- Bạn có **tabular features** (như whale footprint, OHLCV)
- Cần **interpretability** (giải thích được tại sao model quyết định)
- Muốn **iterate nhanh** (train nhanh, thử nhiều experiments)

---

### 2.3 Nhóm 2: Deep Learning (Neural Networks)

#### 🧠 Cách hoạt động (Giải thích đơn giản)

Neural network mô phỏng cách não người xử lý thông tin:
- **Input layer**: Nhận features (giá, volume, shark values...)
- **Hidden layers**: Xử lý và tìm patterns phức tạp
- **Output layer**: Đưa ra dự đoán

#### 📊 Các loại Deep Learning cho Time Series

##### a) LSTM (Long Short-Term Memory)

```
Candle 1 → Candle 2 → Candle 3 → Candle 4 → Candle 5 → Prediction
    ↓         ↓         ↓         ↓         ↓
   [=========== MEMORY (nhớ thông tin quan trọng) ===========]
```

**Cách hoạt động:**
- LSTM có "bộ nhớ" để nhớ thông tin từ quá khứ
- 3 cổng (gates) quyết định: nhớ gì, quên gì, output gì
- Tốt cho việc tìm patterns trong chuỗi dài

**Ưu điểm:**
- Capture được temporal dependencies (pattern theo thời gian)
- Tự động học features từ raw data
- Tốt khi có nhiều data (>10,000 samples)

**Nhược điểm:**
- Chậm hơn tree-based models 10-100x
- Cần nhiều data và computing power
- Black box - khó giải thích
- Dễ overfit nếu data ít

##### b) Transformer (Self-Attention)

```
┌─────────────────────────────────────────────────────────┐
│ "Mỗi candle nhìn vào TẤT CẢ các candles khác để        │
│  quyết định cái nào quan trọng cho prediction"          │
│                                                         │
│ Candle 1  Candle 2  Candle 3  Candle 4  Candle 5       │
│     ↑         ↑         ↑         ↑         ↑          │
│     └─────────┴─────────┴─────────┴─────────┘          │
│              ATTENTION: Cái nào quan trọng?             │
└─────────────────────────────────────────────────────────┘
```

**Cách hoạt động:**
- Không xử lý tuần tự như LSTM
- Mỗi time step "attend" (chú ý) đến tất cả time steps khác
- Tìm ra relationships bất kỳ đâu trong sequence

**Ưu điểm:**
- Capture long-range dependencies tốt hơn LSTM
- Song song hóa được (train nhanh hơn LSTM)
- State-of-the-art cho nhiều bài toán NLP/time series

**Nhược điểm:**
- Cần RẤT NHIỀU data (100,000+ samples lý tưởng)
- Complex architecture, khó tune
- Overfitting dễ xảy ra với data ít

##### c) TCN (Temporal Convolutional Network)

**Cách hoạt động:**
- Dùng convolution thay vì recurrence
- Dilated convolutions để capture long-range patterns
- Song song hóa tốt hơn LSTM

**Ưu điểm:**
- Nhanh hơn LSTM
- Không bị vanishing gradient
- Tốt cho fixed-length sequences

---

### 2.4 Nhóm 3: Reinforcement Learning (RL)

#### 🎮 Cách hoạt động (Giải thích đơn giản)

Thay vì dự đoán giá, RL học trực tiếp **cách trading**:

```
┌─────────────────────────────────────────────────────────┐
│                  REINFORCEMENT LEARNING                  │
│                                                         │
│   State (Features)  →  Agent  →  Action (LONG/SHORT)   │
│         ↑                              ↓                │
│         └──── Reward (Profit/Loss) ←───┘                │
│                                                         │
│   Agent học từ trial-and-error:                         │
│   • Action tốt → Reward dương → Làm nhiều hơn          │
│   • Action xấu → Reward âm → Tránh đi                  │
└─────────────────────────────────────────────────────────┘
```

#### 📊 Các thuật toán RL phổ biến

| Algorithm | Đặc điểm | Phù hợp cho |
|-----------|----------|-------------|
| **DQN** | Q-learning với neural network | Discrete actions (Buy/Sell/Hold) |
| **PPO** | Policy gradient, stable training | Continuous actions |
| **A2C/A3C** | Actor-Critic, parallel training | Large-scale training |

#### ✅ Ưu điểm

```
✓ Học trực tiếp strategy, không chỉ prediction
✓ Tự động tính đến transaction costs, risk
✓ Có thể optimize cho metrics cuối cùng (Sharpe, PnL)
✓ Không cần labeled data (self-supervised)
```

#### ❌ Nhược điểm

```
✗ Cực kỳ khó train - unstable, sensitive to hyperparameters
✗ Cần RẤT NHIỀU data và computing power
✗ Dễ overfit vào historical patterns
✗ Black box - không biết tại sao agent làm gì
✗ Không phù hợp cho pilot/MVP
```

---

### 2.5 Nhóm 4: Ensemble & Hybrid

#### 🔗 Cách hoạt động

Kết hợp nhiều models để tận dụng ưu điểm của từng loại:

```
┌─────────────────────────────────────────────────────────┐
│                    ENSEMBLE METHODS                      │
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ LightGBM │    │   LSTM   │    │ XGBoost  │          │
│  │ (Tabular)│    │ (Temporal)│   │ (Robust) │          │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘          │
│       ↓               ↓               ↓                 │
│  ┌────────────────────────────────────────────┐        │
│  │            STACKING/VOTING                  │        │
│  │   Combine predictions → Final Decision      │        │
│  └────────────────────────────────────────────┘        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### Các phương pháp Ensemble

| Method | Cách kết hợp |
|--------|-------------|
| **Voting** | Mỗi model vote, majority wins |
| **Averaging** | Trung bình predictions của các models |
| **Stacking** | Meta-model học cách combine các base models |
| **Blending** | Weighted average với weights học được |

---

### 2.6 So sánh tổng hợp tất cả Models

| Criteria | Tree-Based | LSTM/GRU | Transformer | RL | Ensemble |
|----------|------------|----------|-------------|-----|----------|
| **Min Data** | 30 ngày | 90 ngày | 180+ ngày | 360+ ngày | 60 ngày |
| **Train Speed** | ⚡ Nhanh | 🐢 Chậm | 🐢 Chậm | 🐌 Rất chậm | 🐢 Chậm |
| **Inference** | ⚡ <1ms | 🐢 10-50ms | 🐢 50-100ms | ⚡ <10ms | 🐢 Tổng các models |
| **Accuracy*** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| **Interpretable** | ✅ Có | ❌ Không | ❌ Không | ❌ Không | 🔶 Phần nào |
| **Overfit Risk** | Thấp | Cao | Rất cao | Rất cao | Trung bình |
| **Complexity** | Thấp | Cao | Rất cao | Cực cao | Cao |

*\*Accuracy phụ thuộc nhiều vào data và feature engineering*

---

## 3. Considerations & Answers (Updated)

### Data Availability

```
✅ Training data: >180 ngày (~10,800 samples)
✅ Backtest data: 90 ngày (~5,400 samples)
✅ Total: ~270 ngày (~16,200 samples)
```

Với lượng data này, bạn có thể thử nghiệm hầu hết các models (trừ pure Transformer và RL).

---

### Q1: Nên lựa chọn model nào để có thể đưa ra được cả 3 hành động trên?

### 🎯 Recommendation: LightGBM + LSTM Ensemble

Với 180+ ngày data, tôi recommend **Ensemble approach**:

```
┌─────────────────────────────────────────────────────────┐
│               RECOMMENDED ARCHITECTURE                   │
│                                                         │
│   Features ───┬───→ LightGBM ────┐                      │
│               │                   │                      │
│               │                   ├───→ Meta-Model ──→ Prediction
│               │                   │      (LightGBM)      │
│               └───→ LSTM ────────┘                      │
│                                                         │
│   LightGBM: Capture tabular feature patterns            │
│   LSTM: Capture temporal/sequential patterns            │
│   Meta-Model: Learn optimal combination                 │
└─────────────────────────────────────────────────────────┘
```

### Phân tích chi tiết

| Tiêu chí | LightGBM only | LSTM only | **Ensemble (Recommended)** |
|----------|---------------|-----------|---------------------------|
| Tabular features | ⭐⭐⭐ Xuất sắc | ⭐ Kém | ⭐⭐⭐ Xuất sắc |
| Temporal patterns | ⭐ Kém | ⭐⭐⭐ Xuất sắc | ⭐⭐⭐ Xuất sắc |
| Train speed | ⭐⭐⭐ Nhanh | ⭐ Chậm | ⭐⭐ Trung bình |
| Accuracy potential | ⭐⭐ Tốt | ⭐⭐ Tốt | ⭐⭐⭐ Cao nhất |
| Robustness | ⭐⭐ Tốt | ⭐ Dễ overfit | ⭐⭐⭐ Rất tốt |
| Implementation | ⭐⭐⭐ Dễ | ⭐⭐ Trung bình | ⭐⭐ Trung bình |

### Tuy nhiên, để tối ưu thời gian, đề xuất pilot strategy:

```
┌─────────────────────────────────────────────────────────┐
│                   PILOT STRATEGY                         │
│                                                         │
│  Phase 1 (Tuần 1): LightGBM only                        │
│  ├── Quick baseline                                     │
│  ├── Feature importance analysis                        │
│  └── Validate data pipeline                             │
│                                                         │
│  Phase 2 (Tuần 2): Add LSTM                             │
│  ├── Train LSTM on sequence data                        │
│  └── Compare with LightGBM                              │
│                                                         │
│  Phase 3 (Tuần 3): Ensemble                             │
│  ├── Stacking: LightGBM + LSTM                          │
│  └── Final evaluation                                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Cách chuyển từ Prediction → 3 Actions:**

```python
# Model predict: price_change_percent (liên tục)
predicted_change = ensemble_model.predict(features)  # e.g., +0.5%, -0.3%

# Convert to actions với dynamic threshold
LONG_THRESHOLD = 0.3    # Tune trên validation set
SHORT_THRESHOLD = -0.3  # Tune trên validation set

if predicted_change > LONG_THRESHOLD:
    action = "LONG"      # Vào vị thế và dự đoán Giá tăng
    confidence = (predicted_change - LONG_THRESHOLD) / LONG_THRESHOLD
elif predicted_change < SHORT_THRESHOLD:
    action = "SHORT"     # Vào vị thế và dự đoán Giá giảm
    confidence = abs(predicted_change - SHORT_THRESHOLD) / abs(SHORT_THRESHOLD)
else:
    action = "HOLD"      # Không vào vị thế
    confidence = 0
```

**Ưu điểm của Ensemble approach:**
- Thresholds có thể tune sau khi train
- Có thể scale position size dựa vào confidence
- LightGBM capture whale footprint patterns tốt
- LSTM capture momentum/trend patterns
- Meta-model học cách kết hợp tối ưu

---

### Q2: Số lượng ngày cần dùng để train?

**Với data availability của bạn:**

```
Total: 270 ngày (~16,200 candles)

Data Split đề xuất:
├── Train: 180 ngày (67%) ─────→ ~10,800 samples
├── Validation: 45 ngày (17%) ─→ ~2,700 samples
└── Test/Backtest: 45 ngày (17%) → ~2,700 samples

Hoặc sử dụng Walk-Forward:
├── Initial Train: 120 ngày
├── Walk-Forward Window: 30 ngày
└── Test: 30 ngày (rolling)
```

**Lưu ý quan trọng:**
- Phải split theo thời gian (không random shuffle) để tránh data leakage
- Validation set dùng để tune hyperparameters & thresholds
- Test set chỉ dùng 1 lần cuối cùng để đánh giá final model

---

## 3. Available Data Summary

**Data Source:** `stock_trading_feature_candles` table (symbol="VN30")

### 3.1 Price Features (OHLCV)
| Feature | Description | Unit |
|---------|-------------|------|
| `open`, `high`, `low`, `close` | VN30 Index OHLCV | Index points |
| `volume` | Total volume 30 stocks | Shares |
| `value` | Total traded value | Million VND |

### 3.2 Whale Footprint Features
| Feature | Description |
|---------|-------------|
| `vn30_shark450_buy_value` | Giá trị mua của cá mập (≥450M) |
| `vn30_shark450_sell_value` | Giá trị bán của cá mập (≥450M) |
| `vn30_shark900_buy_value` | Giá trị mua của cá mập lớn (≥900M) |
| `vn30_shark900_sell_value` | Giá trị bán của cá mập lớn (≥900M) |
| `vn30_shark450_buy_ratio_5d_pc` | Ratio vs 5-day baseline |
| `vn30_percent_shark450_buy_sell` | % mua trong tổng flow cá mập |
| `vn30_shark450_urgency_spread` | VWAP urgency indicator |

---

## 5. Implementation Plan (Updated for Ensemble)

### Phase 1: Data Preparation (2-3 ngày)

**Mục tiêu:** Chuẩn bị dataset cho cả LightGBM và LSTM

1. **Export VN30 features từ DB**
   ```python
   # Query từ stock_trading_feature_candles
   # Filter: symbol="VN30", interval=300
   # Range: 270 ngày data
   ```

2. **Tạo Target Variable (Y)**
   ```python
   # Primary: Next candle return (for regression)
   y_regression = (next_close - current_close) / current_close * 100
   
   # Secondary: Direction classification (for validation)
   y_classification = 1 if y_regression > 0.3 else (-1 if y_regression < -0.3 else 0)
   ```

3. **Feature Engineering cho LightGBM**
   ```python
   # Lag features (point-in-time values from previous candles)
   for col in ['close', 'volume', 'vn30_shark450_buy_value', ...]:
       for lag in [1, 2, 3, 5, 10]:
           df[f'{col}_lag_{lag}'] = df[col].shift(lag)
   
   # Returns
   for lag in [1, 5, 10, 20]:
       df[f'return_{lag}'] = df['close'].pct_change(lag) * 100
   
   # Rolling statistics
   for window in [5, 10, 20]:
       df[f'close_ma_{window}'] = df['close'].rolling(window).mean()
       df[f'volume_std_{window}'] = df['volume'].rolling(window).std()
   
   # Time features
   df['hour'] = df['time'].dt.hour
   df['minute'] = df['time'].dt.minute
   df['candle_of_day'] = (df['time'] - df['time'].dt.normalize()).dt.seconds // 300
   ```

4. **Feature Engineering cho LSTM**
   ```python
   # LSTM cần sequences, không cần lag features
   # Normalize features to [0, 1] or standardize
   
   sequence_length = 20  # 20 candles lookback (~100 minutes)
   features_for_lstm = ['close', 'volume', 'vn30_shark450_buy_value', ...]
   
   # Create sequences: shape (samples, sequence_length, num_features)
   X_lstm = create_sequences(df[features_for_lstm], sequence_length)
   ```

5. **Data Split**
   ```
   Total: 270 ngày (~16,200 candles)
   ├── Train: Day 1-180 (~10,800 samples)
   ├── Validation: Day 181-225 (~2,700 samples)
   └── Test: Day 226-270 (~2,700 samples)
   ```

---

### Phase 2: LightGBM Baseline (3-4 ngày)

**Mục tiêu:** Train LightGBM và establish baseline performance

1. **Train LightGBM**
   ```python
   import lightgbm as lgb
   
   params = {
       "objective": "regression",
       "metric": "mae",
       "num_leaves": 63,
       "learning_rate": 0.03,
       "feature_fraction": 0.8,
       "bagging_fraction": 0.8,
       "bagging_freq": 5,
       "min_child_samples": 20,
       "lambda_l1": 0.1,
       "lambda_l2": 0.1,
   }
   
   model_lgb = lgb.train(
       params,
       train_data,
       valid_sets=[val_data],
       num_boost_round=1000,
       callbacks=[
           lgb.early_stopping(100),
           lgb.log_evaluation(50),
       ],
   )
   ```

2. **Hyperparameter Tuning với Optuna**
   ```python
   import optuna
   
   def objective(trial):
       params = {
           "num_leaves": trial.suggest_int("num_leaves", 20, 150),
           "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.1),
           "feature_fraction": trial.suggest_float("feature_fraction", 0.5, 1.0),
           ...
       }
       # Train and return validation MAE
       return validation_mae
   
   study = optuna.create_study(direction="minimize")
   study.optimize(objective, n_trials=100)
   ```

3. **Feature Importance Analysis**
   ```python
   importance = model_lgb.feature_importance(importance_type='gain')
   # Visualize top 20 features
   # Remove features with near-zero importance
   ```

4. **Baseline Metrics**
   - MAE, RMSE trên validation set
   - Directional accuracy
   - Quick backtest để check viability

---

### Phase 3: LSTM Model (4-5 ngày)

**Mục tiêu:** Train LSTM để capture temporal patterns

1. **LSTM Architecture**
   ```python
   import torch
   import torch.nn as nn
   
   class LSTMPredictor(nn.Module):
       def __init__(self, input_size, hidden_size=64, num_layers=2, dropout=0.2):
           super().__init__()
           self.lstm = nn.LSTM(
               input_size=input_size,
               hidden_size=hidden_size,
               num_layers=num_layers,
               batch_first=True,
               dropout=dropout,
           )
           self.fc = nn.Sequential(
               nn.Linear(hidden_size, 32),
               nn.ReLU(),
               nn.Dropout(0.2),
               nn.Linear(32, 1),  # Predict price change %
           )
       
       def forward(self, x):
           lstm_out, _ = self.lstm(x)
           last_hidden = lstm_out[:, -1, :]  # Take last timestep
           return self.fc(last_hidden)
   ```

2. **Training Loop**
   ```python
   model = LSTMPredictor(input_size=len(features), hidden_size=128)
   optimizer = torch.optim.AdamW(model.parameters(), lr=0.001, weight_decay=0.01)
   scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(optimizer, patience=10)
   
   for epoch in range(100):
       train_loss = train_epoch(model, train_loader)
       val_loss = validate(model, val_loader)
       scheduler.step(val_loss)
       
       # Early stopping
       if early_stopper.should_stop(val_loss):
           break
   ```

3. **Hyperparameter Tuning**
   - Hidden size: [64, 128, 256]
   - Num layers: [1, 2, 3]
   - Sequence length: [10, 20, 30, 50]
   - Dropout: [0.1, 0.2, 0.3]

4. **Compare với LightGBM**
   - So sánh MAE, directional accuracy
   - Analyze cases where LSTM > LightGBM và ngược lại

---

### Phase 4: Ensemble (3-4 ngày)

**Mục tiêu:** Combine LightGBM + LSTM để maximize accuracy

1. **Stacking Architecture**
   ```python
   # Level 1: Base models
   pred_lgb = model_lgb.predict(X_val_lgb)      # LightGBM predictions
   pred_lstm = model_lstm.predict(X_val_lstm)   # LSTM predictions
   
   # Level 2: Meta-model
   meta_features = np.column_stack([
       pred_lgb,
       pred_lstm,
       # Optionally: original features
   ])
   
   meta_model = lgb.LGBMRegressor()
   meta_model.fit(meta_features_train, y_train)
   
   # Final prediction
   final_pred = meta_model.predict(meta_features_test)
   ```

2. **Alternative: Weighted Average**
   ```python
   # Learn optimal weights on validation set
   def find_optimal_weights(pred_lgb, pred_lstm, y_true):
       best_weight = 0.5
       best_mae = float('inf')
       
       for w in np.arange(0.1, 0.9, 0.05):
           combined = w * pred_lgb + (1-w) * pred_lstm
           mae = mean_absolute_error(y_true, combined)
           if mae < best_mae:
               best_mae = mae
               best_weight = w
       
       return best_weight
   
   # Typical result: LightGBM 60-70%, LSTM 30-40%
   ```

3. **Threshold Optimization**
   ```python
   def optimize_thresholds(predictions, y_true, metric='sharpe'):
       best_thresholds = (0.3, -0.3)
       best_metric = -float('inf')
       
       for long_th in np.arange(0.1, 0.6, 0.05):
           for short_th in np.arange(-0.6, -0.1, 0.05):
               signals = get_signals(predictions, long_th, short_th)
               returns = calculate_returns(signals, y_true)
               
               if metric == 'sharpe':
                   score = calculate_sharpe(returns)
               elif metric == 'profit_factor':
                   score = calculate_profit_factor(returns)
               
               if score > best_metric:
                   best_metric = score
                   best_thresholds = (long_th, short_th)
       
       return best_thresholds
   ```

---

### Phase 5: Backtesting (3-4 ngày)

**Mục tiêu:** Validate strategy với realistic trading simulation

1. **Trading Simulator**
   ```python
   class TradingSimulator:
       def __init__(self, initial_capital=100_000_000):  # 100M VND
           self.capital = initial_capital
           self.position = 0  # -1, 0, 1
           self.entry_price = 0
           self.trades = []
           
           # Hyperparameters (tunable)
           self.take_profit = 0.5  # 0.5%
           self.stop_loss = 0.3    # 0.3%
           self.transaction_cost = 0.0003  # 0.03%
       
       def step(self, prediction, current_price, long_th, short_th):
           # Exit logic first
           if self.position != 0:
               pnl = (current_price - self.entry_price) / self.entry_price
               pnl *= self.position  # Adjust for short
               
               if pnl >= self.take_profit or pnl <= -self.stop_loss:
                   self._close_position(current_price)
           
           # Entry logic
           if self.position == 0:
               if prediction > long_th:
                   self._open_position(1, current_price)  # LONG
               elif prediction < short_th:
                   self._open_position(-1, current_price)  # SHORT
   ```

2. **Walk-Forward Validation**
   ```python
   # Rolling window training
   window_size = 120  # 120 days training
   step_size = 20     # Re-train every 20 days
   
   results = []
   for start in range(0, len(data) - window_size - 30, step_size):
       train_end = start + window_size
       test_end = train_end + 30
       
       # Train on window
       model = train_ensemble(data[start:train_end])
       
       # Test on next 30 days
       test_result = backtest(model, data[train_end:test_end])
       results.append(test_result)
   
   # Analyze stability across windows
   ```

3. **Metrics Dashboard**
   | Metric | Target | Description |
   |--------|--------|-------------|
   | Win Rate | > 52% | % trades profitable |
   | Profit Factor | > 1.3 | Gross profit / Gross loss |
   | Max Drawdown | < 8% | Largest peak-to-trough |
   | Sharpe Ratio | > 1.5 | Risk-adjusted return |
   | Avg Trades/Day | 5-15 | Not too few, not too many |
   | Avg Holding Time | 15-60 min | Intraday only |

---

### Phase 6: Production Pipeline (4-5 ngày)

**Mục tiêu:** Deploy model để chạy realtime

```
┌─────────────────────────────────────────────────────────────────┐
│                    PRODUCTION ARCHITECTURE                       │
│                                                                 │
│  Every 5 minutes:                                               │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ VN30Feature  │───→│ Feature      │───→│ Ensemble     │      │
│  │ Pipeline     │    │ Engineering  │    │ Model        │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                 │               │
│                                                 ↓               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ Order        │←───│ Position     │←───│ Signal       │      │
│  │ Executor     │    │ Manager      │    │ Generator    │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│        │                                                        │
│        ↓                                                        │
│  ┌──────────────┐                                              │
│  │ VN30F1M      │                                              │
│  │ Exchange     │                                              │
│  └──────────────┘                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Project Timeline (Updated)

| Phase | Task | Duration | Output |
|-------|------|----------|--------|
| **1** | Data Preparation | 2-3 ngày | Clean dataset, feature engineering |
| **2** | LightGBM Baseline | 3-4 ngày | Tuned LightGBM model |
| **3** | LSTM Model | 4-5 ngày | Trained LSTM model |
| **4** | Ensemble | 3-4 ngày | Stacked ensemble model |
| **5** | Backtesting | 3-4 ngày | Performance report, walk-forward results |
| **6** | Production Pipeline | 4-5 ngày | Realtime prediction system |
| | **Total** | **19-25 ngày (~4-5 tuần)** | |

---

## 7. Success Criteria

| Criteria | Minimum | Target | Measurement |
|----------|---------|--------|-------------|
| Directional Accuracy | > 52% | > 55% | % đúng hướng trên test set |
| MAE | < 0.5% | < 0.35% | Mean Absolute Error |
| Win Rate | > 50% | > 55% | % trades profitable |
| Profit Factor | > 1.2 | > 1.5 | Gross profit / Gross loss |
| Sharpe Ratio | > 1.0 | > 1.5 | Annualized risk-adjusted return |
| Max Drawdown | < 12% | < 8% | Largest peak-to-trough |
| Walk-forward Consistency | > 60% | > 75% | % windows profitable |

---

## 8. Next Steps (Immediate)

1. [ ] Verify data trong DB: count số ngày, check missing dates
2. [ ] Setup project structure cho AI package (metan-ai hoặc trong stock package)
3. [ ] Implement DataLoader class để fetch và prepare data
4. [ ] Create Jupyter notebook cho EDA và feature exploration
5. [ ] Install dependencies: lightgbm, torch, optuna, pandas

---

## 9. Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Overfitting | Cao | Cao | Walk-forward validation, regularization |
| Data leakage | Trung bình | Cao | Strict time-based splits, careful feature engineering |
| Model degradation | Cao | Trung bình | Daily monitoring, periodic retraining |
| Market regime change | Cao | Cao | Ensemble diversification, regime detection |
| Slippage/execution | Trung bình | Trung bình | Conservative sizing, limit orders |

---

## 10. Future Enhancements (Post-Pilot)

1. **Additional Models**
   - TCN (Temporal Convolutional Network)
   - Attention-based LSTM
   - XGBoost as another base model

2. **Additional Features**
   - VN30F1M premium/discount vs VN30 spot
   - Bid-ask spread, order book imbalance
   - Sector rotation signals
   - Macro indicators (USD/VND, oil price)

3. **Advanced Risk Management**
   - Dynamic position sizing based on confidence
   - Correlation with VIX/market conditions
   - Kelly criterion position sizing
   - Regime-aware trading rules
