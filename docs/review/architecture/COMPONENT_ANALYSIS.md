# LightGBM Component Analysis

## Overview

This document provides a detailed breakdown of each major component in the LightGBM codebase, including responsibilities, interfaces, and implementation details.

---

## 1. Core Components

### 1.1 Boosting Module

**Location**: `/home/user/LightGBM/src/boosting/`

**Purpose**: Implements gradient boosting algorithms that orchestrate the training process.

#### Key Classes

##### Boosting (Interface)
**File**: `/home/user/LightGBM/include/LightGBM/boosting.h`

```cpp
class Boosting {
 public:
  virtual void Init(const Config* config, const Dataset* train_data,
                    const ObjectiveFunction* objective_function,
                    const std::vector<const Metric*>& training_metrics) = 0;
  virtual bool TrainOneIter(const score_t* gradients,
                            const score_t* hessians) = 0;
  virtual void RefitTree(const std::vector<std::vector<int>>& leaf_preds) = 0;
  virtual double* GetTrainingScore(int64_t* out_len) = 0;
  virtual void PredictRaw(const double* features, double* output,
                          const PredictionEarlyStopInstance*) const = 0;
  virtual void Predict(const double* features, double* output,
                       const PredictionEarlyStopInstance*) const = 0;

  // Factory method
  static Boosting* CreateBoosting(const std::string& type, const char* filename);
};
```

##### GBDT (Implementation)
**File**: `/home/user/LightGBM/src/boosting/gbdt.cpp`

**Members**:
| Member | Type | Purpose |
|--------|------|---------|
| `iter_` | int | Current iteration |
| `train_data_` | const Dataset* | Training dataset pointer |
| `tree_learner_` | unique_ptr<TreeLearner> | Tree learning strategy |
| `objective_function_` | const ObjectiveFunction* | Loss function |
| `models_` | vector<unique_ptr<Tree>> | Trained decision trees |
| `train_score_updater_` | unique_ptr<ScoreUpdater> | Score management |
| `data_sample_strategy_` | unique_ptr<SampleStrategy> | Sampling strategy |

**Key Methods**:
- `Init()`: Initialize training infrastructure (lines 59-300)
- `TrainOneIter()`: Single iteration training (lines 400-500)
- `Bagging()`: Data sampling implementation
- `UpdateScore()`: Update prediction scores
- `RollbackOneIter()`: Support for early stopping

#### Boosting Variants

| Variant | File | Description |
|---------|------|-------------|
| GBDT | `gbdt.cpp` | Standard gradient boosting |
| DART | `dart.cpp` | Dropouts for preventing overfitting |
| GOSS | via `sample_strategy.cpp` | Gradient-based one-side sampling |
| RF | `rf.cpp` | Random forest variant |

---

### 1.2 TreeLearner Module

**Location**: `/home/user/LightGBM/src/treelearner/`

**Purpose**: Learns individual decision trees using histogram-based split finding.

#### Interface
**File**: `/home/user/LightGBM/include/LightGBM/tree_learner.h`

```cpp
class TreeLearner {
 public:
  virtual void Init(const Dataset* train_data, bool is_constant_hessian) = 0;
  virtual Tree* Train(const score_t* gradients, const score_t* hessians,
                      bool is_first_tree) = 0;
  virtual void SetBaggingData(const Dataset* subset,
                              const data_size_t* used_indices,
                              data_size_t num_data) = 0;

  static TreeLearner* CreateTreeLearner(const std::string& learner_type,
                                        const std::string& device_type,
                                        const Config* config,
                                        const bool boosting_on_cuda);
};
```

#### Implementations

##### SerialTreeLearner (CPU)
**File**: `/home/user/LightGBM/src/treelearner/serial_tree_learner.cpp`

**Algorithm Flow**:
```
1. BeforeTrain() - Prepare for iteration
2. ConstructHistograms() - Build feature histograms
3. FindBestSplits() - Find optimal splits per leaf
4. Split() - Partition data
5. Repeat until max_leaves or min_data_in_leaf
```

**Key Data Structures**:
- `smaller_leaf_splits_`: LeafSplits for smaller leaf
- `larger_leaf_splits_`: LeafSplits for larger leaf
- `histogram_pool_`: Pool of histogram bins

##### Parallel Variants

| Implementation | File | Strategy |
|---------------|------|----------|
| DataParallelTreeLearner | `data_parallel_tree_learner.cpp` | AllReduce histograms across workers |
| FeatureParallelTreeLearner | `feature_parallel_tree_learner.cpp` | Distribute features across workers |
| VotingParallelTreeLearner | `voting_parallel_tree_learner.cpp` | Vote on best splits |

##### GPU Variants

| Implementation | File | Backend |
|---------------|------|---------|
| GPUTreeLearner | `gpu_tree_learner.cpp` | OpenCL via Boost.Compute |
| CUDASingleGPUTreeLearner | `cuda/cuda_single_gpu_tree_learner.cpp` | CUDA |

---

### 1.3 Dataset Module

**Location**: `/home/user/LightGBM/src/io/`

**Purpose**: Manages training and validation data, including loading, binning, and storage.

#### Dataset Class
**File**: `/home/user/LightGBM/include/LightGBM/dataset.h` (lines 487-1071)

**Responsibilities**:
1. Store binned feature values
2. Manage metadata (labels, weights, queries)
3. Provide efficient data access patterns
4. Support multiple input formats

**Key Members**:
```cpp
class Dataset {
 private:
  std::vector<std::unique_ptr<FeatureGroup>> feature_groups_;
  std::vector<int> used_feature_map_;
  int num_features_;
  int num_total_features_;
  data_size_t num_data_;
  Metadata metadata_;
  std::vector<std::string> feature_names_;
  // CUDA support
  #ifdef USE_CUDA
  std::unique_ptr<CUDAColumnData> cuda_column_data_;
  #endif
};
```

#### Metadata Class
**File**: `/home/user/LightGBM/include/LightGBM/dataset.h` (lines 48-397)

**Stores**:
- Labels (`std::vector<label_t>`)
- Sample weights
- Initial scores (init_score)
- Query boundaries (for ranking)
- Query weights

#### DatasetLoader Class
**File**: `/home/user/LightGBM/src/io/dataset_loader.cpp`

**Supported Formats**:
- LibSVM (sparse)
- TSV/CSV (dense)
- Binary (pre-processed)
- Arrow/Parquet (via Arrow integration)

**Loading Pipeline**:
```
Raw File -> Parser -> Sampling -> BinMapper -> FeatureGroup -> Dataset
```

#### FeatureGroup Class
**File**: `/home/user/LightGBM/include/LightGBM/feature_group.h`

**Purpose**: Groups features for efficient storage and histogram building

**Features**:
- Multi-bin mapping for categorical features
- Sparse vs dense storage decision
- Cache-line aligned storage

---

### 1.4 Objective Module

**Location**: `/home/user/LightGBM/src/objective/`

**Purpose**: Computes gradients and hessians for optimization.

#### Interface
**File**: `/home/user/LightGBM/include/LightGBM/objective_function.h`

```cpp
class ObjectiveFunction {
 public:
  virtual void Init(const Metadata& metadata, data_size_t num_data) = 0;
  virtual void GetGradients(const double* score,
                            score_t* gradients, score_t* hessians) const = 0;
  virtual const char* GetName() const = 0;
  virtual bool IsConstantHessian() const { return false; }
  virtual double BoostFromScore(int class_id) const { return 0.0; }
  virtual void ConvertOutput(const double* input, double* output) const;

  static ObjectiveFunction* CreateObjectiveFunction(const std::string& type,
                                                    const Config& config);
};
```

#### Implementations

| Objective | File | Purpose |
|-----------|------|---------|
| `RegressionL2loss` | `regression_objective.hpp` | MSE regression |
| `RegressionL1loss` | `regression_objective.hpp` | MAE regression |
| `RegressionHuberLoss` | `regression_objective.hpp` | Huber loss |
| `RegressionQuantileloss` | `regression_objective.hpp` | Quantile regression |
| `BinaryLogloss` | `binary_objective.hpp` | Binary classification |
| `MulticlassOVA` | `multiclass_objective.hpp` | One-vs-All multiclass |
| `MulticlassSoftmax` | `multiclass_objective.hpp` | Softmax multiclass |
| `LambdaRank` | `rank_objective.hpp` | Learning to rank |
| `RankXENDCG` | `xentropy_objective.hpp` | Cross-entropy NDCG |

#### CUDA Objectives
**Location**: `/home/user/LightGBM/src/objective/cuda/`

GPU-accelerated versions maintain interface compatibility while computing gradients on device.

---

### 1.5 Metric Module

**Location**: `/home/user/LightGBM/src/metric/`

**Purpose**: Evaluates model performance during training.

#### Interface
**File**: `/home/user/LightGBM/include/LightGBM/metric.h`

```cpp
class Metric {
 public:
  virtual void Init(const Metadata& metadata, data_size_t num_data) = 0;
  virtual const std::vector<std::string>& GetName() const = 0;
  virtual double factor_to_bigger_better() const = 0;
  virtual std::vector<double> Eval(const double* score,
                                   const ObjectiveFunction* objective) const = 0;

  static Metric* CreateMetric(const std::string& type, const Config& config);
};
```

#### Implementations

| Metric | File | Description |
|--------|------|-------------|
| RMSE | `regression_metric.hpp` | Root mean squared error |
| MAE | `regression_metric.hpp` | Mean absolute error |
| AUC | `binary_metric.hpp` | Area under ROC curve |
| Logloss | `binary_metric.hpp` | Logarithmic loss |
| Multi-logloss | `multiclass_metric.hpp` | Multi-class log loss |
| NDCG | `rank_metric.hpp` | Normalized DCG |
| MAP | `rank_metric.hpp` | Mean average precision |

---

## 2. Infrastructure Components

### 2.1 Network Module

**Location**: `/home/user/LightGBM/src/network/`

**Purpose**: Provides distributed communication primitives.

#### Network Class (Static)
**File**: `/home/user/LightGBM/include/LightGBM/network.h`

**Key Methods**:
```cpp
class Network {
 public:
  static void Init(Config config);
  static void Dispose();
  static int rank();
  static int num_machines();

  // Collective operations
  static void Allreduce(char* input, comm_size_t input_size, int type_size,
                        char* output, const ReduceFunction& reducer);
  static void Allgather(char* input, comm_size_t send_size, char* output);
  static void ReduceScatter(char* input, comm_size_t input_size, int type_size,
                            const comm_size_t* block_start,
                            const comm_size_t* block_len, char* output,
                            comm_size_t output_size,
                            const ReduceFunction& reducer);
};
```

#### Communication Algorithms

##### BruckMap (All-gather)
- O(log n) communication steps
- O(all_size) total data transferred

##### RecursiveHalvingMap (Reduce-scatter)
- Handles non-power-of-2 machine counts
- Leader election for odd groups

#### Linkers (Communication Backends)

| Implementation | File | Description |
|---------------|------|-------------|
| Linkers | `linkers_socket.cpp` | TCP/IP socket communication |
| Linkers (MPI) | `linkers_mpi.cpp` | MPI collective operations |

---

### 2.2 CUDA Module

**Location**: `/home/user/LightGBM/src/cuda/`

**Purpose**: GPU-accelerated operations for NVIDIA GPUs.

#### Core CUDA Components

| Component | File | Purpose |
|-----------|------|---------|
| CUDAColumnData | `cuda_column_data.cu` | GPU storage for feature data |
| CUDAMetadata | `cuda_metadata.cu` | GPU storage for metadata |
| CUDAHistogramConstructor | `cuda_histogram_constructor.cu` | GPU histogram building |
| CUDABestSplitFinder | `cuda_best_split_finder.cu` | GPU split finding |
| CUDADataPartition | `cuda_data_partition.cu` | GPU data partitioning |
| CUDAScoreUpdater | `cuda_score_updater.cu` | GPU score management |

#### Memory Management

```cpp
// From /home/user/LightGBM/include/LightGBM/cuda/cuda_utils.h
template <typename T>
void AllocateCUDAMemory(T** out_ptr, size_t size, const char* file, int line);

template <typename T>
void CopyFromHostToCUDADevice(T* dst, const T* src, size_t size);

template <typename T>
void CopyFromCUDADeviceToHost(T* dst, const T* src, size_t size);
```

---

### 2.3 Configuration System

**Location**: `/home/user/LightGBM/src/io/config.cpp`

#### Config Struct
**File**: `/home/user/LightGBM/include/LightGBM/config.h`

**Categories**:
1. **Core Parameters**: `num_iterations`, `learning_rate`, `num_leaves`
2. **IO Parameters**: `data`, `valid`, `input_model`, `output_model`
3. **Objective Parameters**: `objective`, `num_class`
4. **Tree Parameters**: `max_depth`, `min_data_in_leaf`
5. **Boosting Parameters**: `boosting`, `bagging_fraction`
6. **Device Parameters**: `device_type`, `gpu_device_id`
7. **Network Parameters**: `num_machines`, `local_listen_port`

#### Parameter Aliasing
**File**: `/home/user/LightGBM/src/io/config.cpp`

Supports multiple names for the same parameter:
```cpp
// Example aliases
{"num_iterations", "num_iteration", "n_iter", "num_tree", ...}
{"learning_rate", "shrinkage_rate", "eta", ...}
```

---

### 2.4 Utility Libraries

**Location**: `/home/user/LightGBM/include/LightGBM/utils/`

| Utility | File | Purpose |
|---------|------|---------|
| Threading | `threading.h` | Thread management utilities |
| OpenMP Wrapper | `openmp_wrapper.h` | OpenMP abstraction |
| Log | `log.h` | Logging infrastructure |
| Common | `common.h` | Common utilities, string parsing |
| Text Reader | `text_reader.h` | Efficient file reading |
| Random | `random.h` | Random number generation |
| Array Args | `array_args.h` | Array utilities |
| Byte Buffer | `byte_buffer.h` | Binary serialization |
| Chunked Array | `chunked_array.hpp` | Growing arrays |

---

## 3. Language Binding Components

### 3.1 C API

**File**: `/home/user/LightGBM/src/c_api.cpp`

**Architecture**:
```
External Caller -> C API (c_api.h) -> C++ Implementation -> Return Code
                        |
                   Error Handling (LGBM_GetLastError)
```

**Handle Types**:
```cpp
typedef void* DatasetHandle;
typedef void* BoosterHandle;
typedef void* FastConfigHandle;
```

**Error Handling Pattern**:
```cpp
int LGBM_SomeFunction(...) {
  API_BEGIN();
  // ... implementation
  API_END();
}

// API_BEGIN/API_END macros handle exceptions -> error codes
```

### 3.2 Python Package

**Location**: `/home/user/LightGBM/python-package/lightgbm/`

#### Module Structure

| Module | Purpose | Key Classes |
|--------|---------|-------------|
| `basic.py` | Core C API wrapper | `Booster`, `Dataset` |
| `sklearn.py` | sklearn integration | `LGBMClassifier`, `LGBMRegressor` |
| `dask.py` | Distributed training | `DaskLGBMClassifier` |
| `engine.py` | Training orchestration | `train()`, `cv()` |
| `callback.py` | Callbacks | `early_stopping`, `log_evaluation` |
| `plotting.py` | Visualization | `plot_importance`, `plot_tree` |

#### ctypes Integration (basic.py)

```python
# Library loading
_LIB = ctypes.CDLL(lib_path)

# Function signatures
_LIB.LGBM_DatasetCreateFromMat.restype = ctypes.c_int
_LIB.LGBM_DatasetCreateFromMat.argtypes = [
    ctypes.c_void_p,
    ctypes.c_int,
    ...
]
```

### 3.3 R Package

**Location**: `/home/user/LightGBM/R-package/`

**Structure**:
```
R-package/
|-- R/                    # R wrapper functions
|   |-- lightgbm.R
|   |-- lgb.Booster.R
|   |-- lgb.Dataset.R
|   |-- lgb.importance.R
|-- src/
|   |-- lightgbm_R.cpp    # C++ glue code
|   |-- lightgbm_R.h
|-- man/                  # Documentation
|-- DESCRIPTION
|-- NAMESPACE
```

### 3.4 SWIG (Java)

**Location**: `/home/user/LightGBM/swig/`

**Interface File**: `lightgbmlib.i`

**Key Components**:
- Type mappings for Java primitives
- String array handling
- Arrow array extensions
- Memory management directives

---

## 4. Component Interactions

### 4.1 Training Pipeline

```
Application.InitTrain()
    |
    +-> Boosting::CreateBoosting()
    |       |
    |       +-> GBDT() or DART() or RF()
    |
    +-> ObjectiveFunction::CreateObjectiveFunction()
    |       |
    |       +-> BinaryLogloss() or RegressionL2loss() etc.
    |
    +-> Application.LoadData()
            |
            +-> DatasetLoader::LoadFromFile()
                    |
                    +-> Parser::CreateParser()
                    +-> Dataset::Construct()
                    +-> FeatureGroup binning

Training Loop:
    GBDT::TrainOneIter()
        |
        +-> SampleStrategy::Bagging()
        +-> ObjectiveFunction::GetGradients()
        +-> TreeLearner::Train()
        |       |
        |       +-> ConstructHistograms()
        |       +-> FindBestSplits()
        |       +-> Split()
        |       +-> -> Tree
        |
        +-> ScoreUpdater::AddScore()
        +-> Metric::Eval()
```

### 4.2 Prediction Pipeline

```
Booster::Predict() / LGBM_BoosterPredictForMat()
    |
    +-> Load model trees
    +-> For each data point:
    |       |
    |       +-> Tree::Predict() for each tree
    |       +-> Sum predictions
    |       +-> ObjectiveFunction::ConvertOutput()
    |
    +-> Return predictions
```

### 4.3 Distributed Training

```
Worker 0                    Worker 1                    Worker N
    |                           |                           |
    +-- LoadData(rank=0) -------+-- LoadData(rank=1) ------+-- LoadData(rank=N)
    |                           |                           |
    +-- ConstructHistograms ----+-- ConstructHistograms ---+-- ConstructHistograms
    |                           |                           |
    +==============  Network::Allreduce(histograms)  ===============+
    |                           |                           |
    +-- FindBestSplit ----------+-- FindBestSplit ---------+-- FindBestSplit
    |                           |                           |
    +==============  Network::Allgather(split_info)  ===============+
    |                           |                           |
    +-- ApplySplit -------------+-- ApplySplit ------------+-- ApplySplit
```

---

## 5. Component Metrics

### 5.1 Code Size Analysis

| Component | Lines of Code | Files |
|-----------|--------------|-------|
| boosting/ | ~8,000 | 15 |
| treelearner/ | ~12,000 | 25 |
| io/ | ~15,000 | 30 |
| objective/ | ~3,000 | 10 |
| metric/ | ~2,500 | 8 |
| network/ | ~2,500 | 6 |
| cuda/ | ~15,000 | 40 |
| c_api.cpp | ~3,000 | 1 |

### 5.2 Dependency Graph (Simplified)

```
                    +-------------+
                    | Application |
                    +------+------+
                           |
              +------------+------------+
              |                         |
       +------v------+          +-------v-------+
       |   Boosting  |          | DatasetLoader |
       +------+------+          +-------+-------+
              |                         |
    +---------+---------+       +-------v-------+
    |         |         |       |    Dataset    |
+---v---+ +---v---+ +---v---+   +-------+-------+
|TreeLrn| |Objectv| |Metric |           |
+---+---+ +-------+ +-------+   +-------v-------+
    |                           | FeatureGroup  |
+---v---+                       +---------------+
|Network|
+-------+
```

---

*This component analysis is part of the LightGBM architecture review documentation.*
