# Marimo + QCodes: 反應式實驗量測與分析平台

<div align="center">

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![QCodes Compatible](https://img.shields.io/badge/QCodes-0.55+-green.svg)](https://github.com/microsoft/Qcodes)
[![Marimo Compatible](https://img.shields.io/badge/Marimo-0.8+-purple.svg)](https://marimo.io)

**將實驗量測控制與數據分析完美融合的開源平台**

[快速開始](#快速開始) • [文檔](#文檔) • [示例](#示例) • [貢獻](#貢獻) • [許可證](#許可證)

</div>

---

## 概述

**Marimo + QCodes** 是一個革命性的實驗量測平台，整合了：

- 🎯 **Marimo** 的反應式 Python 筆記本框架
- 🔬 **QCodes** 的量子儀器量測控制系統
- ⚡ 實時交互式 UI 與自動化數據分析
- 📊 強大的可視化與統計工具

### 核心特點

| 特點 | 優勢 |
|------|------|
| **直觀界面** | 研究員無需編程知識即可控制儀器 |
| **實時反饋** | 邊測邊看，快速調整參數 |
| **完全可重現** | 自動保存完整的參數和元數據 |
| **即時可視化** | Plotly 交互式圖表實時更新 |
| **一鍵部署** | 輕鬆分享為 Web 應用 |
| **開源協作** | MIT 許可，社區驅動 |

---

## 快速開始

### 最低需求

- Python 3.10 或更高版本
- pip 或 conda

### 安裝

#### 方法 1: 從 PyPI 安裝（推薦）

```bash
pip install marimo-qcodes
```

#### 方法 2: 從源代碼安裝

```bash
git clone https://github.com/yourusername/marimo-qcodes.git
cd marimo-qcodes
pip install -e .
```

#### 方法 3: 使用 Docker

```bash
docker pull marimo-qcodes:latest
docker run -p 8080:8080 -v $(pwd)/data:/data marimo-qcodes:latest
```

### 驗證安裝

```python
import marimo as mo
import qcodes as qc
from marimo_qcodes import QCoDesMarimo

print(f"✓ Marimo 版本: {mo.__version__}")
print(f"✓ QCodes 版本: {qc.__version__}")
print(f"✓ QCoDesMarimo 已就緒")
```

### 最小化示例

```python
# first_experiment.py
import marimo as mo
from marimo_qcodes import QCoDesMarimo

app = mo.App()

# 初始化控制器
controller = QCoDesMarimo("My Experiment")

# 為參數生成 UI
voltage_slider = controller.register_parameter(
    name="Gate Voltage",
    min_val=-5.0,
    max_val=5.0,
    step=0.1
)

# 執行掃描
def measurement_func(voltage):
    # 您的測量邏輯
    return voltage ** 2

sweep_data = controller.create_sweep(
    param_name="Gate Voltage",
    measured_func=measurement_func,
    start=-5.0,
    stop=5.0,
    num_points=100
)

# 顯示結果
mo.vstack([
    mo.md("# 實驗結果"),
    voltage_slider,
    mo.ui.table(sweep_data)
])
```

運行應用：

```bash
marimo edit first_experiment.py      # 編輯模式，實時更新
marimo run first_experiment.py        # 應用模式
marimo run --host 0.0.0.0 first_experiment.py  # 部署為 Web 服務
```

---

## 文檔

### 📖 完整文檔

我們提供了詳盡的文檔，涵蓋所有方面：

| 文檔 | 內容 | 閱讀時間 |
|------|------|--------|
| [**快速開始指南**](docs/quick-start-guide.md) | 安裝、第一個應用、常見錯誤 | 15 分鐘 |
| [**完整技術方案**](docs/marimo-qcodes-integration.md) | 架構設計、代碼實現、最佳實踐 | 2 小時 |
| [**API 參考**](docs/api-reference.md) | 所有類和方法的詳細文檔 | 1 小時 |
| [**用戶指南**](docs/user-guide.md) | 逐步教程和實際示例 | 1.5 小時 |
| [**開發者指南**](docs/developer-guide.md) | 貢獻、擴展、測試框架 | 1 小時 |

### 🎓 學習路徑

**初學者** (0-2 小時)
1. ✓ 閱讀本 README 的核心概念部分
2. ✓ 跟隨「快速開始指南」設置環境
3. ✓ 運行 `examples/01_simple_measurement.py`

**中級使用者** (2-6 小時)
1. ✓ 閱讀「用戶指南」的前 5 章
2. ✓ 運行 `examples/02_parameter_sweep.py` 和 `03_real_time_monitoring.py`
3. ✓ 自己修改示例進行實驗

**高級開發者** (6+ 小時)
1. ✓ 深入學習「完整技術方案」
2. ✓ 研究「API 參考」的核心類
3. ✓ 閱讀「開發者指南」
4. ✓ 為項目做出貢獻

---

## 核心概念

### 架構概覽

```
┌─────────────────────────────────────────────────────┐
│  Marimo UI 層                                        │
│  ┌────────────┬────────────┬────────────────────┐   │
│  │  滑塊      │  下拉選單  │  實時圖表           │   │
│  │  數值輸入  │  複選框    │  數據表格           │   │
│  └────────────┴────────────┴────────────────────┘   │
└────────────────────┬─────────────────────────────────┘
                     │ 用戶交互
┌────────────────────▼─────────────────────────────────┐
│  Marimo 反應式執行層                                  │
│  自動依賴跟蹤 → 級聯更新 → 實時可視化               │
└────────────────────┬─────────────────────────────────┘
                     │ 參數變更
┌────────────────────▼─────────────────────────────────┐
│  QCoDesMarimo 橋接層 (核心)                           │
│  ┌──────────────┬──────────────┬──────────────┐      │
│  │ 參數UI生成   │ 數據訂閱管理 │ 掃描進度跟蹤 │      │
│  │ 雙向數據同步 │ 錯誤處理     │ 自動校準     │      │
│  └──────────────┴──────────────┴──────────────┘      │
└────────────────────┬─────────────────────────────────┘
                     │ 設置/讀取
┌────────────────────▼─────────────────────────────────┐
│  QCodes 量測控制層                                   │
│  Parameter • Instrument • Station • Measurement     │
└────────────────────┬─────────────────────────────────┘
                     │ 命令/數據
┌────────────────────▼─────────────────────────────────┐
│  硬件層                                               │
│  電壓源、測量儀、溫度控制器、其他儀器設備            │
└─────────────────────────────────────────────────────┘
```

### 工作流程

```
用戶移動滑塊 
  ↓
Marimo 檢測變化 
  ↓
QCoDesMarimo 橋接層處理
  ↓
參數值發送到 QCodes
  ↓
儀器接收並執行命令
  ↓
儀器返回測量值
  ↓
QCodes 存儲數據
  ↓
橋接層觸發 UI 更新
  ↓
Plotly 圖表實時更新
```

---

## 示例

### 📁 示例應用結構

```
examples/
├── 01_simple_measurement.py          # 簡單電壓測量
├── 02_parameter_sweep.py             # 1D 參數掃描
├── 03_real_time_monitoring.py        # 實時監測
├── 04_quantum_dot.py                 # 量子點特性測量
├── 05_temperature_dependence.py      # 溫度依賴性研究
├── 06_adaptive_sampling.py           # 自適應採樣
├── 07_multi_instrument.py            # 多儀器協調
└── 08_remote_collaboration.py        # 遠程協作
```

### 運行示例

```bash
# 編輯模式（開發）
marimo edit examples/01_simple_measurement.py

# 應用模式（演示）
marimo run examples/02_parameter_sweep.py

# 部署為 Web 服務
marimo run --host 0.0.0.0 --port 8080 examples/04_quantum_dot.py
```

### 示例 1: 簡單掃描

```python
# examples/simple_sweep.py
import marimo as mo
import numpy as np
from marimo_qcodes import QCoDesMarimo

app = mo.App()

# 初始化
controller = QCoDesMarimo("Simple Gate Sweep")

# 執行掃描（模擬測量）
def measure(voltage):
    return np.sin(voltage) + np.random.normal(0, 0.05)

data = controller.create_sweep(
    param_name="Gate Voltage",
    measured_func=measure,
    start=-2 * np.pi,
    stop=2 * np.pi,
    num_points=200
)

# 顯示結果
mo.vstack([
    mo.md("# 門電壓掃描"),
    controller.create_control_panel(),
    mo.md("## 結果"),
    mo.ui.table(data),
])
```

---

## 主要功能

### 🎛️ UI 控制

```python
# 自動為 QCodes 參數生成 UI
voltage_slider = controller.register_parameter(
    name="Gate Voltage",
    min_val=-10.0,
    max_val=10.0,
    step=0.1,
    widget_type="slider"
)

current_slider = controller.register_parameter(
    name="Bias Current",
    min_val=0,
    max_val=100,
    step=1,
    widget_type="number"
)

mode = controller.register_parameter(
    name="Measurement Mode",
    options=["DC", "AC Lock-in", "Pulsed"],
    widget_type="dropdown"
)
```

### 📊 掃描引擎

```python
# 1D 掃描
data_1d = controller.perform_1d_sweep(
    sweep_param=gate_voltage,
    measured_param=lockin.ch1,
    start=-5.0,
    stop=5.0,
    num_points=100
)

# 2D 掃描
data_2d = controller.perform_2d_sweep(
    sweep_param1=gate_voltage,
    sweep_param2=bias_voltage,
    measured_param=lockin.ch1,
    start1=-5.0, stop1=5.0, num1=50,
    start2=0, stop2=10.0, num2=50
)

# 多維掃描
data_nd = controller.perform_nd_sweep(
    sweep_params=[gate, bias, temperature],
    measured_param=signal,
    ranges=[(-5, 5), (0, 10), (4.2, 300)],
    num_points=[30, 30, 10]
)
```

### 📈 實時監測

```python
# 實時訂閱數據
def on_measurement_update(data):
    print(f"新數據點: {data}")

controller.subscribe_to_measurement(on_measurement_update)

# 啟動實時監測
monitoring_results = controller.monitor_signal(
    parameter=lockin.ch1,
    duration_seconds=60,
    poll_interval_ms=100,
    update_callback=update_plot
)
```

### 📊 可視化

```python
from marimo_qcodes.visualization import ExperimentVisualizer

# 1D 圖表
fig_1d = ExperimentVisualizer.plot_1d_sweep(
    data_1d,
    x_label="Gate Voltage (V)",
    y_label="Conductance (2e²/h)"
)

# 2D 熱力圖
fig_2d = ExperimentVisualizer.plot_2d_heatmap(
    data_2d,
    x_coords=gate_values,
    y_coords=bias_values,
    title="Conductance Map"
)

# 時間序列
fig_ts = ExperimentVisualizer.plot_time_series(
    time_data,
    title="Lock-in Signal vs Time"
)

# 儀表板
dashboard = ExperimentVisualizer.create_dashboard({
    "1D Sweep": fig_1d,
    "2D Map": fig_2d,
    "Time Series": fig_ts
})
```

### 💾 數據管理

```python
from marimo_qcodes.data_management import ExperimentReporter

reporter = ExperimentReporter("My Experiment")

# 自動保存數據和圖表
reporter.add_measurement(
    name="gate_sweep",
    data=data_1d,
    figure=fig_1d
)

# 導出為多種格式
reporter.export_csv("results/gate_sweep.csv")
reporter.export_hdf5("results/experiment.h5")
reporter.export_json("results/metadata.json")

# 生成實驗報告
report_path = reporter.generate_report()
```

### 🔧 自動校準

```python
from marimo_qcodes.calibration import AutomaticCalibration

calibrator = AutomaticCalibration(controller)

# 校準 DAC 輸出
dac_calib = calibrator.calibrate_reference_voltage(
    dac_param=dac.voltage,
    ref_meter=multimeter,
    test_points=[0, 1, 5, 10]
)

# 應用校準
corrected_value = calibrator.apply_calibration(
    raw_value=5.0,
    calibration_type='dac_calibration'
)
```

---

## 應用案例

### 🔬 量子點特性測量

```python
# examples/quantum_dot.py
# 實時測量量子點的能級結構和庫侖阻斷效應

controller = QCoDesMarimo("Quantum Dot Characterization")

# 配置參數
gate_ui = controller.register_parameter("Gate Voltage", -2.0, 2.0)
bias_ui = controller.register_parameter("Bias Voltage", 0, 5.0)

# 執行 2D 掃描
conductance_map = controller.perform_2d_sweep(...)

# 識別峰值（量子點簽名）
peaks = find_peaks_in_conductance(conductance_map)

# 繪製及分析
plot_with_peak_annotations(conductance_map, peaks)
```

### 🌡️ 溫度依賴性研究

```python
# examples/temperature_dependence.py
# 研究物性隨溫度的變化

# 自動掃描溫度並進行完整特性測量
for temperature in np.linspace(4.2, 300, 20):
    cryostat.set_temperature(temperature)
    time.sleep(300)  # 等待溫度穩定
    
    # 對每個溫度進行完整掃描
    data = controller.perform_1d_sweep(...)
    
    # 儲存結果
    reporter.add_measurement(f"sweep_T{temperature}K", data)
```

### 🎯 實驗優化與標定

```python
# examples/auto_calibration.py
# 自動化校準與最優化流程

# 多點校準
calib_coeffs = calibrator.calibrate_multi_point(...)

# 應用校準
sweep_data_corrected = apply_calibration_to_sweep(sweep_data, calib_coeffs)

# 對比校準前後
comparison_plot = create_before_after_comparison(
    sweep_data, 
    sweep_data_corrected
)
```

---

## 項目結構

```
marimo-qcodes/
├── README.md                          # 本文件
├── LICENSE                            # MIT 許可證
├── setup.py / pyproject.toml          # 包配置
├── requirements.txt                   # 依賴列表
│
├── marimo_qcodes/                     # 主模塊
│   ├── __init__.py
│   ├── bridge.py                      # QCoDesMarimo 橋接類
│   ├── ui.py                          # UI 組件生成
│   ├── sweeps.py                      # 掃描引擎
│   ├── visualization.py               # 可視化工具
│   ├── data_management.py             # 數據管理
│   ├── subscription.py                # 實時訂閱系統
│   ├── calibration.py                 # 校準工具
│   └── utils.py                       # 實用函數
│
├── examples/                          # 示例應用
│   ├── 01_simple_measurement.py
│   ├── 02_parameter_sweep.py
│   ├── 03_real_time_monitoring.py
│   ├── 04_quantum_dot.py
│   ├── 05_temperature_dependence.py
│   ├── 06_adaptive_sampling.py
│   ├── 07_multi_instrument.py
│   └── 08_remote_collaboration.py
│
├── tests/                             # 測試套件
│   ├── test_bridge.py
│   ├── test_sweeps.py
│   ├── test_visualization.py
│   ├── test_data_management.py
│   └── conftest.py
│
├── docs/                              # 文檔
│   ├── quick-start-guide.md
│   ├── marimo-qcodes-integration.md
│   ├── api-reference.md
│   ├── user-guide.md
│   ├── developer-guide.md
│   ├── faq.md
│   └── troubleshooting.md
│
├── docker/                            # Docker 配置
│   ├── Dockerfile
│   └── docker-compose.yml
│
└── scripts/                           # 實用腳本
    ├── setup_environment.sh
    ├── run_tests.sh
    └── build_docker.sh
```

---

## 系統要求

### 最低配置
- Python 3.10 或更高版本
- 4 GB RAM
- 2 GB 硬盤空間

### 推薦配置
- Python 3.11+
- 8 GB RAM
- SSD（加快數據寫入）
- Linux 或 macOS（Windows 需要 WSL2）

### 儀器連接
- GPIB 卡（National Instruments 或相容）
- USB-GPIB 適配器
- 以太網 SCPI 儀器支持

---

## 依賴關係

### 核心依賴
```
marimo >= 0.8.0          # 反應式筆記本框架
QCodes >= 0.55.0         # 量子儀器控制
numpy >= 1.24.0          # 數值計算
pandas >= 2.0.0          # 數據處理
scipy >= 1.10.0          # 科學計算
plotly >= 5.14.0         # 交互式繪圖
```

### 可選依賴
```
pyvisa >= 1.13.0         # GPIB/USB 通訊
pyvisa-py >= 0.6.0       # 純 Python VISA
h5py >= 3.0.0            # HDF5 數據存儲
altair >= 5.0.0          # 聲明式可視化
```

### 開發依賴
```
pytest >= 7.0            # 單元測試
pytest-cov >= 4.0        # 覆蓋率報告
black >= 23.0            # 代碼格式化
flake8 >= 6.0            # 代碼檢查
mypy >= 1.0              # 類型檢查
sphinx >= 6.0            # 文檔生成
```

---

## 安裝詳細說明

### 虛擬環境設置（推薦）

```bash
# 創建虛擬環境
python -m venv marimo-qcodes-env

# 激活虛擬環境
# Linux/macOS:
source marimo-qcodes-env/bin/activate
# Windows:
marimo-qcodes-env\Scripts\activate

# 升級 pip
pip install --upgrade pip

# 安裝軟件包
pip install marimo-qcodes

# 驗證安裝
python -c "from marimo_qcodes import QCoDesMarimo; print('✓ Installation successful!')"
```

### Conda 環境設置

```bash
# 創建 conda 環境
conda create -n marimo-qcodes python=3.11 -y

# 激活環境
conda activate marimo-qcodes

# 安裝依賴
conda install -c conda-forge marimo qcodes numpy pandas plotly -y

# 從 PyPI 安裝本包
pip install marimo-qcodes
```

### Docker 設置

```bash
# 構建 Docker 映像
docker build -t marimo-qcodes:latest -f docker/Dockerfile .

# 運行容器
docker run -d \
  --name marimo-experiment \
  -p 8080:8080 \
  -v $(pwd)/data:/data \
  -e QCODES_DB_PATH=/data/qcodes.db \
  marimo-qcodes:latest

# 訪問應用
# 打開瀏覽器: http://localhost:8080
```

---

## 使用方法

### 基本工作流程

1. **初始化控制器**
   ```python
   from marimo_qcodes import QCoDesMarimo
   controller = QCoDesMarimo("My Experiment")
   ```

2. **註冊儀器參數**
   ```python
   voltage = controller.register_parameter("Gate V", -10, 10, 0.1)
   ```

3. **執行掃描**
   ```python
   data = controller.perform_1d_sweep(...)
   ```

4. **可視化結果**
   ```python
   fig = ExperimentVisualizer.plot_1d_sweep(data)
   ```

5. **導出數據**
   ```python
   reporter.export_csv("results.csv")
   ```

### 實際應用模板

```python
# template_experiment.py
import marimo as mo
from marimo_qcodes import QCoDesMarimo
from marimo_qcodes.visualization import ExperimentVisualizer

app = mo.App()

# 初始化
controller = QCoDesMarimo("My Experiment")

# UI 定義
mo.vstack([
    mo.md("# 實驗控制面板"),
    controller.create_control_panel(),
    
    mo.md("## 參數選擇"),
    mo.hstack([
        mo.md("開始值:"),
        start_val := mo.ui.number(start=0, value=0)
    ]),
    mo.hstack([
        mo.md("結束值:"),
        end_val := mo.ui.number(start=10, value=10)
    ]),
    mo.hstack([
        mo.md("點數:"),
        num_points := mo.ui.number(start=10, stop=1000, value=100)
    ]),
])

# 執行掃描
data = controller.perform_1d_sweep(
    sweep_param=gate_voltage,
    measured_param=detector,
    start=start_val.value,
    stop=end_val.value,
    num_points=int(num_points.value)
)

# 可視化
mo.vstack([
    mo.md("## 結果"),
    ExperimentVisualizer.plot_1d_sweep(data),
    mo.md("### 統計"),
    mo.md(f"平均值: {data['signal'].mean():.4f}")
])
```

---

## 常見問題 (FAQ)

### Q: 我需要真實的硬件才能使用嗎？

**A:** 不需要！所有示例都使用模擬儀器，無需實際硬件。但如果您有儀器，代碼完全兼容。

### Q: 支持哪些儀器？

**A:** 任何 QCodes 支持的儀器都可以使用。詳見 [QCodes 驅動列表](https://github.com/QCoDeS/Qcodes_drivers)。

### Q: 如何連接我的自定義儀器？

**A:** 參考「開發者指南」編寫自定義 QCodes 驅動程序。

### Q: 性能如何？能否進行快速掃描？

**A:** 取決於儀器和通訊速度。通常可達 10-100 點/秒。查看「性能優化指南」了解詳情。

### Q: 可以在 Windows 上使用嗎？

**A:** 是的，但推薦使用 WSL2 或 Anaconda 環境以獲得最佳相容性。

### Q: 如何處理大型數據集？

**A:** 使用 HDF5 格式存儲，支持高效的增量訪問。

更多常見問題見 [完整 FAQ](docs/faq.md)。

---

## 故障排除

### 常見問題

| 問題 | 解決方案 |
|------|--------|
| 導入錯誤 | 運行 `pip install -e .` 重新安裝 |
| VISA 連接失敗 | 安裝 `pyvisa-py`；檢查 GPIB 驅動 |
| UI 無響應 | 檢查 Marimo 版本；清除瀏覽器緩存 |
| 數據保存失敗 | 檢查文件夾權限；確保磁盤有空間 |
| 掃描超時 | 增加儀器延遲時間；檢查通訊速度 |

詳見 [故障排除指南](docs/troubleshooting.md)。

---

## 貢獻

我們歡迎各種形式的貢獻！

### 如何貢獻

1. **Fork** 此版本庫
2. 創建功能分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 開啟 Pull Request

### 貢獻指南

詳見 [CONTRIBUTING.md](CONTRIBUTING.md)。

### 代碼規範

- 遵循 PEP 8 風格指南
- 使用 Black 進行代碼格式化
- 包含單元測試（目標 >90% 覆蓋率）
- 添加相應的文檔和 docstring

### 報告 Bug

使用 [GitHub Issues](https://github.com/yourusername/marimo-qcodes/issues) 報告 bug。

請包含：
- 問題的清晰描述
- 複現步驟
- 預期行為 vs 實際行為
- 您的環境信息（Python 版本、OS 等）
- 相關的代碼片段或日誌

### 功能請求

在 [GitHub Discussions](https://github.com/yourusername/marimo-qcodes/discussions) 提出功能想法。

---

## 開發者指南

### 設置開發環境

```bash
# 克隆版本庫
git clone https://github.com/yourusername/marimo-qcodes.git
cd marimo-qcodes

# 安裝開發依賴
pip install -e ".[dev]"

# 運行測試
pytest tests/ -v --cov=marimo_qcodes

# 代碼檢查
black marimo_qcodes/ examples/
flake8 marimo_qcodes/
mypy marimo_qcodes/
```

### 代碼結構

- **bridge.py**: 核心 QCoDesMarimo 類
- **ui.py**: Marimo UI 組件生成
- **sweeps.py**: 掃描算法實現
- **visualization.py**: 圖表和繪圖
- **data_management.py**: 數據存儲和導出
- **subscription.py**: 實時數據更新機制

### 測試

```bash
# 運行所有測試
pytest

# 運行特定測試
pytest tests/test_bridge.py -v

# 生成覆蓋率報告
pytest --cov=marimo_qcodes --cov-report=html
```

### 構建文檔

```bash
cd docs
make html
open _build/html/index.html
```

---

## 路線圖

### 已完成 ✅
- [x] QCoDesMarimo 核心類
- [x] 基本 1D 掃描功能
- [x] Marimo UI 集成
- [x] Plotly 可視化
- [x] 數據導出功能
- [x] 完整文檔

### 計劃中 🚀
- [ ] 2D/多維掃描優化
- [ ] WebSocket 實時監測
- [ ] 高級校準工具
- [ ] 機器學習集成
- [ ] 雲部署支持
- [ ] Jupyter 筆記本適配器
- [ ] 移動應用支持

### 考慮中 💡
- 硬件自動檢測
- AI 輔助實驗設計
- 協作實驗平台
- 商業許可選項

詳見 [完整路線圖](docs/ROADMAP.md)。

---

## 許可證

本項目採用 **MIT 許可證** - 詳見 [LICENSE](LICENSE) 文件。

### 第三方許可證

- **Marimo**: MIT License
- **QCodes**: MIT License
- **Plotly**: MIT License

---

## 引用

如果您在研究中使用了本項目，請引用：

```bibtex
@software{marimo_qcodes_2025,
  author = {Your Name},
  title = {Marimo + QCodes: Reactive Experimental Measurement Platform},
  url = {https://github.com/yourusername/marimo-qcodes},
  year = {2025},
  note = {Open Source Software}
}
```

---

## 支持

### 獲得幫助

- 📖 [文檔](https://marimo-qcodes.readthedocs.io)
- 💬 [GitHub Discussions](https://github.com/yourusername/marimo-qcodes/discussions)
- 🐛 [Issue Tracker](https://github.com/yourusername/marimo-qcodes/issues)
- 📧 [Email](mailto:support@example.com)

### 相關項目

- [Marimo](https://github.com/marimo-team/marimo) - 反應式 Python 筆記本
- [QCodes](https://github.com/microsoft/Qcodes) - 量子儀器控制
- [Plotly](https://plotly.com/python/) - 交互式可視化
- [Pandas](https://pandas.pydata.org/) - 數據分析

---

## 作者與致謝

### 核心開發團隊
- [Your Name](https://github.com/yourprofile) - 項目發起人
- [Contributor Name](https://github.com/) - 核心開發者

### 致謝
感謝所有貢獻者、測試者和用戶的支持！

特別感謝：
- Marimo 開發團隊的優秀框架
- Microsoft Qcodes 團隊的強大控制系統
- 我們的測試用戶的反饋和建議

---

## 更新日誌

### [v1.0.0] - 2025-12-15 (計劃發布)

**新增**
- QCoDesMarimo 橋接類完整實現
- 1D/2D 掃描引擎
- Plotly 交互式可視化
- 自動校準系統
- 完整的文檔和示例

**改進**
- 性能優化
- 更好的錯誤處理
- 改進的 UI/UX

詳見 [完整更改日誌](CHANGELOG.md)。

---

<div align="center">

**⭐ 如果您覺得這個項目有用，請給個 Star！**

[⬆ 返回頂部](#marimo--qcodes-反應式實驗量測與分析平台)

</div>

---

## 快速連結

| 連結 | 說明 |
|------|------|
| [快速開始](docs/quick-start-guide.md) | 5 分鐘入門教程 |
| [完整文檔](docs/) | 所有詳細文檔 |
| [API 參考](docs/api-reference.md) | 類和方法文檔 |
| [示例應用](examples/) | 8 個完整示例 |
| [常見問題](docs/faq.md) | 問題解答 |
| [故障排除](docs/troubleshooting.md) | 問題診斷 |
| [貢獻指南](CONTRIBUTING.md) | 如何貢獻 |
| [討論區](https://github.com/yourusername/marimo-qcodes/discussions) | 提問和討論 |

---

**最後更新**: 2025 年 12 月 11 日

