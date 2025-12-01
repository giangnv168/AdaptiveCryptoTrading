# FEEDER PYTORCH CONVERSION - NEW SESSION PROMPT

## CONTEXT

We need to convert the Freqtrade feeders (75a, 75b, 75c) from LightGBM to PyTorch to achieve better GPU utilization.

**Current situation:**
- LightGBM model working with Optuna integration ✅
- GPU support enabled but low utilization (5-20%) - normal for tree algorithms
- User wants 80-100% GPU utilization → Need PyTorch (neural network)
- All working files backed up before conversion

## BACKUP LOCATION (for rollback if needed)

```
/home/freqai/freqtrade/backups/before_pytorch_20251130_102245/
```

**Backup includes:**
- `LightGBMQuantileMultiTarget_Optuna.py` - Current working model
- `ProducerStrategyDualQuantile_LONG.py` - Strategy file
- `config_feeder_75a.json`, `config_feeder_75b.json`, `config_feeder_75c.json` - Feeder configs
- Service files

## WHAT WAS ACCOMPLISHED IN PREVIOUS SESSION

### 1. Optuna Integration with LightGBM ✅
- Created `LightGBMQuantileMultiTarget_Optuna` model
- Loads optimized hyperparameters from InfluxDB bucket `OPTUNA_FEEDER75_LONG`
- BTC/USDT:USDT parameters: LR=0.0380, Trees=632, Leaves=102, Depth=12

### 2. Modified Target Calculation ✅
- Changed from `close.shift(-30)` (single endpoint)
- To `close.rolling(30).max()` (maximum close during 30min window)
- Intermediate column: `_max_close_30m` (underscore prefix = not a target)
- Final target: `&-future_close_30m_long` (captures "how much move up")

### 3. GPU Support Enabled ✅
- System: 2× Tesla T4 GPUs (15GB each)
- LightGBM v4.6.0.99 with GPU support
- GPU parameters: `device="gpu"`, `gpu_use_dp=False`, `gpu_platform_id=0`, `gpu_device_id=0`

### 4. Current Architecture

**Feeders:**
- **75a**: BTC/USDT:USDT (LONG) - Currently running with LightGBM + GPU
- **75b**: Other LONG pairs - Stopped to reduce contention
- **75c**: More LONG pairs - Stopped

**Model:** `LightGBMQuantileMultiTarget_Optuna`
- 2 targets:
  - `&-future_close_30m_long` (α=0.75) - Optimistic upside
  - `&-min_low_30m` (α=0.1) - Conservative downside
- Optuna-optimized per-pair hyperparameters
- GPU-accelerated (but limited to 5-20% GPU usage - normal for trees)

## TASK: CONVERT TO PYTORCH FOR BETTER GPU UTILIZATION

### Goal
Create a PyTorch neural network model that:
1. Predicts the same 2 targets (future_close_30m_long, min_low_30m)
2. Uses quantile loss function
3. Integrates with Optuna parameter loading
4. Achieves 80-100% GPU utilization (vs LightGBM's 5-20%)
5. Maintains or improves prediction quality

### System Information
- **Host**: 192.168.3.75 (freqai@192.168.3.75, password: NewHopes@168)
- **GPUs**: 2× Tesla T4 (15GB each)
- **LightGBM version**: 4.6.0.99
- **PyTorch**: Need to check if installed, install if needed
- **FreqAI**: Already has PyTorch base classes

### Implementation Requirements

#### 1. Create PyTorch Model: `PyTorchQuantileMultiTarget_Optuna.py`

**Location:** `/home/freqai/freqtrade/user_data/freqaimodels/PyTorchQuantileMultiTarget_Optuna.py`

**Must include:**
- Inherit from FreqAI's PyTorch base class (`BasePyTorchRegressor` or similar)
- Multi-output architecture (2 targets)
- Quantile loss function:
  - α=0.75 for `future_close_30m_long`
  - α=0.10 for `min_low_30m`
- Load Optuna hyperparameters from InfluxDB (`OPTUNA_FEEDER75_LONG`)
- GPU-optimized training (`device='cuda'`)
- Proper data normalization/scaling

#### 2. Neural Network Architecture

**Suggested structure:**
```python
Input Layer (652 features)
    ↓
Hidden Layer 1 (ReLU)
    ↓
Hidden Layer 2 (ReLU)
    ↓
Hidden Layer 3 (ReLU)
    ↓
Output Layer (2 targets: future_close_30m_long, min_low_30m)
```

**Hyperparameters to optimize with Optuna:**
- `learning_rate`: 0.001 - 0.1
- `num_layers`: 2-5
- `hidden_dim`: 64-512
- `dropout`: 0.0-0.5
- `batch_size`: 32-256

#### 3. Quantile Loss Implementation

```python
def quantile_loss(y_pred, y_true, quantile):
    """
    Quantile loss for neural networks
    
    For quantile α:
    - If (y_true - y_pred) > 0: loss = α * (y_true - y_pred)
    - If (y_true - y_pred) <= 0: loss = (1-α) * (y_pred - y_true)
    """
    error = y_true - y_pred
    return torch.mean(torch.where(error >= 0, quantile * error, (quantile - 1) * error))
```

#### 4. Optuna Integration

**Load from InfluxDB:**
```python
# From bucket: OPTUNA_FEEDER75_LONG
# Measurement: feeder_params
# Fields to load:
# - learning_rate
# - hidden_dim
# - num_layers
# - dropout
# - batch_size
```

**Schema consistency:** Use the same field names as in `feeder_optuna_schema.py`

### Key Differences: LightGBM vs PyTorch

| Aspect | LightGBM (Current) | PyTorch (Target) |
|--------|-------------------|------------------|
| Algorithm | Gradient Boosted Trees | Neural Network |
| GPU Usage | 5-20% (normal for trees) | 80-100% (expected) |
| Training Speed | Moderate | 10-20× faster with GPU |
| Interpretability | High (feature importance) | Low (black box) |
| Data Requirements | 15K samples OK | 15K samples OK (but more is better) |
| Hyperparameters | Trees, leaves, depth | Layers, neurons, dropout |

### Testing Strategy

1. **Create PyTorch model** for feeder 75a
2. **Test training** with BTC/USDT:USDT (single pair)
3. **Monitor GPU utilization** (expect 80-100%)
4. **Compare predictions** with LightGBM
5. **If successful**: Deploy to 75b, 75c
6. **Run both models in parallel** for 1 week
7. **Choose the better performer** based on:
   - Prediction accuracy
   - Trading results
   - Training speed
   - Stability

### FreqAI PyTorch Base Classes

**Check these files for reference:**
```bash
find /home/freqai/freqtrade/freqtrade/freqai -name "*pytorch*.py" -o -name "*PyTorch*.py"
```

**Example models to study:**
- `BasePyTorchRegressor`
- `PyTorchMLPRegressor`
- `PyTorchTransformer` (if exists)

### Critical Requirements (From BOT2 Debugging)

⚠️ **Field name consistency is CRITICAL!**

1. **Use constants from schema:**
```python
from user_data.services.feeder_optuna_schema import (
    FIELD_LEARNING_RATE,
    FIELD_HIDDEN_DIM,
    # etc.
)
```

2. **All values as float:**
```python
point.field("learning_rate", float(params.learning_rate))
```

3. **Filtered InfluxDB queries:**
```flux
from(bucket: "OPTUNA_FEEDER75_LONG")
|> filter(fn: (r) => 
    r._field == "learning_rate" or 
    r._field == "hidden_dim"
)
|> sort(columns: ["_time"], desc: true)
|> limit(n: 1)
```

4. **Validation logging:**
```python
logger.info(f"✅ Loaded Optuna params for {len(params)} pairs")
logger.info(f"🔍 DEBUG: {pair} - lr={lr}, hidden_dim={hd}")
```

### Rollback Plan (If PyTorch Doesn't Work)

```bash
# Restore LightGBM model
cd /home/freqai/freqtrade
cp backups/before_pytorch_20251130_102245/LightGBMQuantileMultiTarget_Optuna.py user_data/freqaimodels/
cp backups/before_pytorch_20251130_102245/ProducerStrategyDualQuantile_LONG.py user_data/strategies/
cp backups/before_pytorch_20251130_102245/config_feeder_75*.json .

# Restart feeders
sudo systemctl restart freqtrade-feeder75a freqtrade-feeder75b freqtrade-feeder75c
```

### Success Criteria

✅ PyTorch model trains successfully  
✅ GPU utilization: 80-100%  
✅ Training speed: 10-20× faster than LightGBM  
✅ Predictions shape: (n_samples, 2)  
✅ Optuna params loading correctly  
✅ No errors in logs  
✅ Model saves and loads properly  

### Expected Challenges

1. **Quantile loss implementation** - Need to implement custom loss
2. **Data normalization** - Neural networks sensitive to scaling
3. **Hyperparameter tuning** - Different params than LightGBM
4. **Overfitting** - Neural networks prone to overfitting on small data
5. **Training stability** - May need learning rate scheduling

### Files to Create/Modify

1. **CREATE**: `/home/freqai/freqtrade/user_data/freqaimodels/PyTorchQuantileMultiTarget_Optuna.py`
2. **MODIFY**: `/home/freqai/freqtrade/config_feeder_75a.json` - Change `freqaimodel` to `PyTorchQuantileMultiTarget_Optuna`
3. **OPTIONALLY CREATE**: Optuna optimizer for PyTorch hyperparameters

### Immediate First Steps

1. Check if PyTorch is installed: `python3 -c "import torch; print(torch.__version__)"`
2. Check FreqAI PyTorch base classes: `find freqtrade/freqai -name "*pytorch*.py"`
3. Study existing PyTorch model structure
4. Create `PyTorchQuantileMultiTarget_Optuna.py` skeleton
5. Implement quantile loss function
6. Add Optuna parameter loading
7. Test with feeder 75a
8. Monitor GPU usage and training progress

---

**Ready to start? Let's create a PyTorch model that leverages those Tesla T4 GPUs properly! 🚀**
