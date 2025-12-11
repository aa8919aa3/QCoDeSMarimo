# 整合 Marimo 與 QCodes 實驗量測控制方案

## 項目概述

本文件詳細說明如何將**Marimo**（反應式Python筆記本應用框架）與**QCodes**（量子儀器量測控制框架）整合，實現一個完整的實驗量測與數據分析解決方案。

### 核心目標

1. **統一界面控制**：使用Marimo的交互式UI組件控制QCodes儀器
2. **即時數據處理**：在Marimo中進行實驗數據的即時分析與動態繪圖
3. **反應式工作流**：利用Marimo的反應式執行特性自動化實驗流程
4. **可部署應用**：將整個實驗系統打包成可共享的Web應用

---

## 第一部分：技術基礎

### 1.1 Marimo 核心特性

#### 反應式執行架構
```
檔案格式: Pure Python (.py)
執行模式: 
  - 互動式筆記本（開發環境）
  - 指令行腳本（批量運行）
  - Web應用（部署分享）

核心特性:
  - 自動化單元格依賴跟蹤
  - UI組件與代碼自動同步
  - 動態繪圖與可視化
  - Markdown嵌入支持
```

#### 主要UI組件
- **控制組件**：滑塊、下拉選單、數值輸入、切換開關
- **數據組件**：表格轉換器、交互式圖表
- **布局組件**：行/列容器、標籤、卡片
- **輸出組件**：Markdown、計算結果、Plotly/Altair圖表

### 1.2 QCodes 核心特性

#### 量測控制架構
```
核心組件:
  - Parameter: 儀器參數（讀、寫、掃描）
  - Instrument: 儀器驅動程序
  - Station: 儀器集合管理
  - Measurement: 量測協議與數據保存
  - DataSet: 量測數據與元數據存儲
```

#### 支持的應用場景
- **參數掃描**：1D、2D、多維掃描
- **實時監測**：數據收集期間即時訂閱與處理
- **自動化控制**：儀器初始化、校準、故障恢復
- **數據管理**：SQLite數據庫、實驗追蹤、重現性

---

## 第二部分：整合架構設計

### 2.1 系統架構圖

```
┌─────────────────────────────────────────────────────────────┐
│                    Marimo 應用界面層                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  控制面板    │  │  監測儀表板  │  │  分析儀表板  │      │
│  │  (UI Widgets)│  │  (Live Plot) │  │  (Charts)    │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
└─────────┼────────────────┼────────────────┼────────────────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│              Marimo 反應式執行層 (Reactive Cell)             │
│  - 參數變更偵測 → 自動觸發相關單元格                          │
│  - UI交互同步 → 變數更新 → 級聯運算                          │
│  - 錯誤處理與狀態管理                                        │
└──────────────────────────┬───────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│           整合橋接層 (Integration Bridge)                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ QCoDesMarimo 類：QCodes ↔ Marimo 雙向綁定             │   │
│  │  - 參數UI生成器                                       │   │
│  │  - 實時數據訂閱管理                                   │   │
│  │  - 掃描進度跟蹤                                       │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────┬───────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│              QCodes 量測控制層                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Measurement  │  │   Parameter  │  │  Instrument  │       │
│  │   Protocol   │  │    Sweep     │  │   Driver     │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└──────────────────────────┬───────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│            硬件連接層（儀器與設備）                           │
│  物理儀器 / 信號處理 / 數據採集                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 數據流

```
【互動式控制流】
UI滑塊移動 → Marimo檢測 → 反應式更新 → QCodes參數設置 → 儀器響應

【監測與反饋流】
儀器測量 → QCodes訂閱 → Marimo實時接收 → 動態圖表更新 → UI反饋

【掃描工作流】
掃描參數配置 → 启动QCodes掃描 → 逐點測量 → 實時數據收集 → 
漸進式分析 → 動態可視化 → 完成後全景分析
```

---

## 第三部分：核心模塊設計

### 3.1 QCoDesMarimo 橋接類

```python
# qcodes_marimo_bridge.py

from dataclasses import dataclass
from typing import Dict, Any, Callable, Optional
import marimo as mo
from qcodes import Parameter, Instrument, Measurement

@dataclass
class ParameterUI:
    """QCodes參數的Marimo UI包裝"""
    parameter: Parameter
    widget: Any  # mo.ui.slider, mo.ui.input等
    label: str
    
    def get_value(self):
        return self.widget.value
    
    def set_value(self, value):
        self.parameter.set(value)

class QCoDesMarimo:
    """QCodes與Marimo的集成控制器"""
    
    def __init__(self, name: str = "QCoDesMarimo Controller"):
        self.name = name
        self.parameters: Dict[str, Parameter] = {}
        self.parameter_uis: Dict[str, ParameterUI] = {}
        self.measurement = None
        self.data_subscribers = []
        self.scan_progress = mo.ui.progress(0, 100)
        
    def register_parameter(self, param: Parameter, 
                          widget_type: str = "slider",
                          **widget_kwargs) -> ParameterUI:
        """註冊參數並生成對應UI組件"""
        # 根據widget_type生成適當的Marimo組件
        if widget_type == "slider":
            widget = mo.ui.slider(
                start=widget_kwargs.get("start", 0),
                stop=widget_kwargs.get("stop", 10),
                step=widget_kwargs.get("step", 0.1),
                label=param.label or param.name
            )
        elif widget_type == "number":
            widget = mo.ui.number(
                start=widget_kwargs.get("start", 0),
                label=param.label or param.name
            )
        # ... 其他widget類型
        
        param_ui = ParameterUI(
            parameter=param,
            widget=widget,
            label=param.label or param.name
        )
        
        self.parameters[param.name] = param
        self.parameter_uis[param.name] = param_ui
        
        return param_ui
    
    def create_control_panel(self) -> mo.Html:
        """生成統一的控制面板UI"""
        controls = []
        for name, param_ui in self.parameter_uis.items():
            controls.append(
                mo.md(f"### {param_ui.label}").left
            )
            controls.append(param_ui.widget)
            controls.append(
                mo.md(f"**當前值**: {param_ui.get_value()}").left
            )
        
        return mo.vstack(controls)
    
    def sync_ui_to_qcodes(self, param_name: str, value: Any):
        """從UI同步值到QCodes參數"""
        if param_name in self.parameters:
            self.parameters[param_name].set(value)
    
    def subscribe_to_measurement(self, 
                                callback: Callable[[Dict[str, Any]], None]):
        """訂閱測量數據"""
        self.data_subscribers.append(callback)
    
    def perform_1d_sweep(self, 
                         sweep_param: Parameter,
                         measured_param: Parameter,
                         start: float, 
                         stop: float, 
                         num_points: int) -> Dict:
        """執行1D參數掃描"""
        
        from qcodes import Station
        station = Station.default
        
        meas = Measurement(name=f"sweep_{sweep_param.name}")
        meas.register_parameter(sweep_param)
        meas.register_parameter(measured_param, setpoints=(sweep_param,))
        
        # 設置進度跟蹤訂閱
        def progress_callback(results_list, length, state):
            progress_pct = (length / num_points) * 100
            self.scan_progress.value = progress_pct
            # 觸發數據訂閱者
            for subscriber in self.data_subscribers:
                subscriber({
                    'progress': progress_pct,
                    'points': length,
                    'data': results_list[-1] if results_list else None
                })
        
        meas.add_subscriber(progress_callback, state={})
        
        data_points = []
        
        with meas.run() as datasaver:
            for i, set_val in enumerate(np.linspace(start, stop, num_points)):
                sweep_param.set(set_val)
                measured_val = measured_param.get()
                
                datasaver.add_result(
                    (sweep_param, set_val),
                    (measured_param, measured_val)
                )
                
                data_points.append({
                    'x': set_val,
                    'y': measured_val
                })
        
        return {
            'dataset': datasaver.dataset,
            'data_points': data_points,
            'completed': True
        }
    
    def get_data_as_dataframe(self):
        """獲取測量數據為Pandas DataFrame"""
        if self.measurement and hasattr(self.measurement, 'dataset'):
            return self.measurement.dataset.to_dataframe()
        return None
```

### 3.2 Marimo 反應式單元格模式

```python
# experiment_notebook.py
# 作為Marimo筆記本執行

import marimo as mo
import numpy as np
import plotly.graph_objects as go
from qcodes_marimo_bridge import QCoDesMarimo

# ============ 初始化 ============
# 在此單元格中定義實驗參數與儀器
_controller = QCoDesMarimo("Quantum Dot Experiment")

# 註冊參數 - 自動生成UI
gate_voltage_ui = _controller.register_parameter(
    param=device.gate_voltage,
    widget_type="slider",
    start=-2.0,
    stop=2.0,
    step=0.01
)

bias_voltage_ui = _controller.register_parameter(
    param=device.bias_voltage,
    widget_type="slider",
    start=0,
    stop=5.0,
    step=0.05
)

# ============ 控制面板單元格 ============
mo.vstack([
    mo.md("# 量子點實驗控制系統"),
    mo.md("## 儀器參數設置"),
    _controller.create_control_panel(),
    mo.md("## 實驗模式選擇"),
    experiment_mode = mo.ui.dropdown(
        options=["1D Sweep", "2D Sweep", "Real-time Monitor"],
        label="選擇實驗類型"
    )
])

# ============ 反應式單元格 - 參數同步 ============
# 當gate_voltage_ui.value變化時，此單元格自動運行
mo.stop(not gate_voltage_ui.value)
_controller.sync_ui_to_qcodes(
    "gate_voltage",
    gate_voltage_ui.value
)

# 當bias_voltage_ui.value變化時，此單元格自動運行
mo.stop(not bias_voltage_ui.value)
_controller.sync_ui_to_qcodes(
    "bias_voltage", 
    bias_voltage_ui.value
)

# ============ 即時監測單元格 ============
@mo.reactive
def real_time_monitor():
    """實時讀取儀器值並更新圖表"""
    import time
    
    readings = []
    timestamps = []
    
    for i in range(100):
        reading = lockin.ch1.get()
        readings.append(reading)
        timestamps.append(time.time())
        time.sleep(0.1)
    
    fig = go.Figure()
    fig.add_trace(go.Scatter(
        x=timestamps,
        y=readings,
        mode='lines+markers',
        name='Lock-in Output'
    ))
    
    return fig

mo.vstack([
    mo.md("## 即時監測"),
    real_time_monitor
])

# ============ 掃描執行單元格 ============
@mo.reactive
def execute_sweep():
    """執行參數掃描"""
    
    if experiment_mode.value == "1D Sweep":
        result = _controller.perform_1d_sweep(
            sweep_param=device.gate_voltage,
            measured_param=lockin.ch1,
            start=-2.0,
            stop=2.0,
            num_points=100
        )
        
        return mo.vstack([
            mo.md("### 掃描進度"),
            _controller.scan_progress,
            mo.md("### 掃描結果"),
            plot_sweep_data(result['data_points'])
        ])
    
    return mo.md("選擇掃描類型以執行")

execute_sweep

# ============ 數據分析單元格 ============
@mo.reactive
def analyze_data():
    """分析掃描數據"""
    
    df = _controller.get_data_as_dataframe()
    
    if df is None:
        return mo.md("暫無數據")
    
    return mo.vstack([
        mo.md("### 數據統計"),
        mo.md(f"**數據點數**: {len(df)}"),
        mo.md(f"**平均值**: {df.iloc[:, -1].mean():.6f}"),
        mo.md(f"**標準差**: {df.iloc[:, -1].std():.6f}"),
        
        mo.md("### 數據表"),
        mo.ui.table(df.head(20)),
        
        mo.md("### 完整圖表"),
        plot_data_analysis(df)
    ])

analyze_data
```

### 3.3 可視化與繪圖模塊

```python
# visualization_module.py

import plotly.graph_objects as go
import pandas as pd
import marimo as mo

class ExperimentVisualizer:
    """實驗數據可視化"""
    
    @staticmethod
    def plot_1d_sweep(data_points: list[dict], 
                     x_label: str = "Gate Voltage (V)",
                     y_label: str = "Current (A)") -> go.Figure:
        """繪製1D掃描結果"""
        
        df = pd.DataFrame(data_points)
        
        fig = go.Figure()
        
        fig.add_trace(go.Scatter(
            x=df['x'],
            y=df['y'],
            mode='lines+markers',
            name='Measurement',
            marker=dict(size=6, color='#1f77b4'),
            line=dict(width=2)
        ))
        
        fig.update_layout(
            title="1D Parameter Sweep",
            xaxis_title=x_label,
            yaxis_title=y_label,
            hovermode='x unified',
            template='plotly_white'
        )
        
        return fig
    
    @staticmethod
    def plot_2d_heatmap(data_2d: np.ndarray,
                       x_coords: np.ndarray,
                       y_coords: np.ndarray,
                       title: str = "2D Scan") -> go.Figure:
        """繪製2D掃描熱力圖"""
        
        fig = go.Figure(data=go.Heatmap(
            z=data_2d,
            x=x_coords,
            y=y_coords,
            colorscale='Viridis'
        ))
        
        fig.update_layout(
            title=title,
            xaxis_title="X Parameter",
            yaxis_title="Y Parameter",
            template='plotly_white'
        )
        
        return fig
    
    @staticmethod
    def plot_time_series(data_points: list[dict],
                        title: str = "Time Series") -> go.Figure:
        """繪製時間序列"""
        
        df = pd.DataFrame(data_points)
        
        fig = go.Figure()
        
        fig.add_trace(go.Scatter(
            x=range(len(df)),
            y=df['value'],
            mode='lines',
            name='Data',
            fill='tozeroy'
        ))
        
        fig.update_layout(
            title=title,
            xaxis_title="Sample Number",
            yaxis_title="Value",
            template='plotly_white'
        )
        
        return fig
    
    @staticmethod
    def create_dashboard(measurements_dict: dict) -> mo.Html:
        """創建多圖表儀表板"""
        
        plots = []
        for name, fig in measurements_dict.items():
            plots.append(mo.md(f"### {name}"))
            plots.append(fig)
        
        return mo.vstack(plots)
```

---

## 第四部分：實現範例

### 4.1 完整實驗範例：量子點特性測量

```python
# 文件: quantum_dot_experiment.py

import marimo as mo
import numpy as np
from pathlib import Path
from datetime import datetime

# 初始化Marimo應用
app = mo.App()

# ============ 第1單元格：導入與配置 ============

__import__('sys').path.insert(0, str(Path.cwd() / 'qcodes_modules'))

import qcodes as qc
from qcodes.instrument_drivers.yokogawa import GS200
from qcodes_marimo_bridge import QCoDesMarimo, ExperimentVisualizer
import pandas as pd

# 設置QCodes日誌與數據庫
qc.config.user.datadir = Path.cwd() / 'qcodes_data'
qc.config.user.datadir.mkdir(exist_ok=True)

# 初始化儀器（示例）
gate_source = GS200("gate", "GPIB0::1::INSTR")  # 門電壓源
bias_source = GS200("bias", "GPIB0::2::INSTR")  # 偏置源
lockin = InstrumentDriver("lockin", "GPIB0::10::INSTR")  # 鎖相放大器

mo.md("""
# 量子點特性測量系統

此應用使用Marimo與QCodes整合實現：
- **儀器控制**：實時設置與掃描
- **數據採集**：自動化測量協議
- **實時分析**：動態數據處理與繪圖
""")

# ============ 第2單元格：控制器初始化 ============

controller = QCoDesMarimo("Quantum Dot Characterization")

# 註冊測量儀器參數
gate_voltage = controller.register_parameter(
    param=gate_source.voltage,
    widget_type="slider",
    start=-2.0,
    stop=2.0,
    step=0.01,
    label="Gate Voltage (V)"
)

bias_voltage = controller.register_parameter(
    param=bias_source.voltage,
    widget_type="slider",
    start=0,
    stop=5.0,
    step=0.05,
    label="Bias Voltage (V)"
)

lockin_sensitivity = controller.register_parameter(
    param=lockin.sensitivity,
    widget_type="dropdown",
    options=['1 nV', '10 nV', '100 nV', '1 µV', '10 µV'],
    label="Lock-in Sensitivity"
)

# ============ 第3單元格：實驗參數配置 ============

mo.vstack([
    mo.md("## 實驗設置"),
    
    mo.md("### 儀器參數"),
    controller.create_control_panel(),
    
    mo.md("### 掃描參數"),
    mo.hstack([
        mo.md("**掃描類型:**"),
        experiment_type := mo.ui.dropdown(
            options=["1D Gate Sweep", "2D Sweep (Gate vs Bias)", 
                    "Real-time Monitor", "Temperature Ramp"],
            label="Select"
        )
    ]),
    
    mo.hstack([
        mo.md("**掃描點數:**"),
        num_points := mo.ui.number(start=10, stop=1000, step=10, value=100)
    ]),
    
    mo.hstack([
        mo.md("**保存路徑:**"),
        save_path := mo.ui.text(value=str(Path.cwd() / 'experiments'))
    ]),
])

# ============ 第4單元格：即時監測 ============

mo.stop(experiment_type.value != "Real-time Monitor")

def real_time_callback(data_dict):
    """實時數據回調"""
    pass

controller.subscribe_to_measurement(real_time_callback)

monitoring_data = []
start_time = datetime.now()

for i in range(500):
    elapsed = (datetime.now() - start_time).total_seconds()
    reading = lockin.ch1.get()
    monitoring_data.append({'time': elapsed, 'value': reading})

mo.vstack([
    mo.md("## 實時監測"),
    ExperimentVisualizer.plot_time_series(
        monitoring_data,
        title="Lock-in Output vs Time"
    )
])

# ============ 第5單元格：掃描執行 ============

mo.stop(experiment_type.value == "Real-time Monitor")

sweep_results = None

if experiment_type.value == "1D Gate Sweep":
    mo.md("### 執行1D掃描...")
    
    sweep_results = controller.perform_1d_sweep(
        sweep_param=gate_source.voltage,
        measured_param=lockin.ch1,
        start=-2.0,
        stop=2.0,
        num_points=int(num_points.value)
    )
    
    # 保存數據
    if sweep_results['completed']:
        df = pd.DataFrame(sweep_results['data_points'])
        export_path = Path(save_path.value) / f"sweep_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
        export_path.parent.mkdir(parents=True, exist_ok=True)
        df.to_csv(export_path)

elif experiment_type.value == "2D Sweep (Gate vs Bias)":
    mo.md("### 執行2D掃描...")
    
    data_2d = np.zeros((int(num_points.value), int(num_points.value)))
    
    for i, gate_val in enumerate(np.linspace(-2.0, 2.0, int(num_points.value))):
        for j, bias_val in enumerate(np.linspace(0, 5.0, int(num_points.value))):
            gate_source.voltage.set(gate_val)
            bias_source.voltage.set(bias_val)
            data_2d[i, j] = lockin.ch1.get()
    
    sweep_results = {
        'data_2d': data_2d,
        'gate_range': (-2.0, 2.0),
        'bias_range': (0, 5.0)
    }

# ============ 第6單元格：結果可視化 ============

mo.stop(sweep_results is None)

if 'data_points' in sweep_results:
    fig = ExperimentVisualizer.plot_1d_sweep(
        sweep_results['data_points'],
        x_label="Gate Voltage (V)",
        y_label="Conductance (2e²/h)"
    )

elif 'data_2d' in sweep_results:
    gate_coords = np.linspace(-2.0, 2.0, sweep_results['data_2d'].shape[0])
    bias_coords = np.linspace(0, 5.0, sweep_results['data_2d'].shape[1])
    
    fig = ExperimentVisualizer.plot_2d_heatmap(
        sweep_results['data_2d'],
        gate_coords,
        bias_coords,
        title="Conductance Map: Gate vs Bias"
    )

mo.vstack([
    mo.md("## 測量結果"),
    fig,
    
    mo.md("### 統計信息"),
    mo.md(f"""
    - **最大值**: {np.max(sweep_results.get('data_2d', np.array([v['y'] for v in sweep_results.get('data_points', [])]))): .6f}
    - **最小值**: {np.min(sweep_results.get('data_2d', np.array([v['y'] for v in sweep_results.get('data_points', [])]))): .6f}
    - **平均值**: {np.mean(sweep_results.get('data_2d', np.array([v['y'] for v in sweep_results.get('data_points', [])]))): .6f}
    """),
])

# ============ 第7單元格：進階分析 ============

mo.stop(sweep_results is None)

analysis_type = mo.ui.dropdown(
    options=["Peak Detection", "Derivative Analysis", 
            "Baseline Subtraction", "FFT Analysis"],
    label="分析類型"
)

mo.md(f"## 進階分析: {analysis_type.value}")

if analysis_type.value == "Peak Detection":
    from scipy.signal import find_peaks
    
    y_data = np.array([p['y'] for p in sweep_results['data_points']])
    peaks, properties = find_peaks(y_data, prominence=0.1)
    
    mo.md(f"偵測到 {len(peaks)} 個峰值")
    mo.md(f"峰值位置: {peaks}")
    mo.md(f"峰值高度: {properties['prominences']}")

# 更多分析選項...
```

### 4.2 系統部署配置

```yaml
# marimo-qcodes-app.yaml
# Docker 與部署配置

version: '3.8'

services:
  marimo-app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - QCODES_DB_PATH=/data/qcodes.db
      - EXPERIMENT_DATA_DIR=/data/experiments
    volumes:
      - ./qcodes_data:/data
      - ./notebooks:/app/notebooks
    command: >
      marimo run 
      --host 0.0.0.0 
      --port 8080
      notebooks/quantum_dot_experiment.py
```

```dockerfile
# Dockerfile

FROM python:3.11-slim

WORKDIR /app

# 安裝系統依賴
RUN apt-get update && apt-get install -y \
    build-essential \
    libhdf5-dev \
    && rm -rf /var/lib/apt/lists/*

# 複製requirement文件
COPY requirements.txt .

# 安裝Python依賴
RUN pip install --upgrade pip && \
    pip install -r requirements.txt

# 複製應用代碼
COPY . .

# 暴露埠
EXPOSE 8080

# 運行應用
CMD ["marimo", "run", "--host", "0.0.0.0", "--port", "8080", "notebooks/experiment.py"]
```

```txt
# requirements.txt

# Marimo
marimo>=0.8.0

# QCodes
QCodes>=0.55.0

# 數據處理
numpy>=1.24.0
pandas>=2.0.0
scipy>=1.10.0

# 可視化
plotly>=5.14.0
altair>=5.0.0

# 儀器驅動（示例）
PyVISA>=1.13.0
PyVISA-py>=0.6.0

# 實用工具
pydantic>=2.0.0
python-dotenv>=1.0.0
```

---

## 第五部分：核心功能實現指南

### 5.1 動態UI生成

```python
# dynamic_ui_generator.py

class DynamicParameterUI:
    """動態為QCodes參數生成Marimo UI"""
    
    @staticmethod
    def create_ui(parameter, widget_config=None):
        """
        根據參數類型自動生成適當的UI組件
        
        Args:
            parameter: QCodes Parameter 對象
            widget_config: 自定義配置字典
        
        Returns:
            Marimo UI 組件
        """
        
        config = widget_config or {}
        param_range = parameter.vals
        
        # 從參數驗證器提取範圍信息
        if hasattr(param_range, 'min_value') and hasattr(param_range, 'max_value'):
            min_val = config.get('start', param_range.min_value)
            max_val = config.get('stop', param_range.max_value)
            
            # 數值範圍參數 → 滑塊
            return mo.ui.slider(
                start=min_val,
                stop=max_val,
                step=config.get('step', (max_val - min_val) / 100),
                label=parameter.label or parameter.name
            )
        
        elif hasattr(param_range, 'values'):
            # 離散選項 → 下拉選單
            return mo.ui.dropdown(
                options=list(param_range.values),
                label=parameter.label or parameter.name
            )
        
        else:
            # 其他情況 → 文本輸入
            return mo.ui.text(
                placeholder=parameter.label or parameter.name
            )
```

### 5.2 實時數據訂閱系統

```python
# realtime_subscription.py

from threading import Thread
from queue import Queue
import time

class RealtimeDataSubscriber:
    """實時數據訂閱與処理"""
    
    def __init__(self, measurement, callback, poll_interval=0.1):
        """
        Args:
            measurement: QCodes Measurement 對象
            callback: 接收數據的回調函數 (data_dict) -> None
            poll_interval: 輪詢間隔（秒）
        """
        self.measurement = measurement
        self.callback = callback
        self.poll_interval = poll_interval
        self.running = False
        self.data_queue = Queue()
        self.thread = None
    
    def start(self):
        """啟動訂閱"""
        self.running = True
        self.thread = Thread(target=self._poll_loop, daemon=True)
        self.thread.start()
    
    def _poll_loop(self):
        """輪詢迴圈"""
        while self.running:
            try:
                # 從測量對象獲取最新數據
                if hasattr(self.measurement, 'dataset'):
                    data = self.measurement.dataset.cache.data
                    self.callback({
                        'timestamp': time.time(),
                        'data': data
                    })
            except Exception as e:
                print(f"Subscription error: {e}")
            
            time.sleep(self.poll_interval)
    
    def stop(self):
        """停止訂閱"""
        self.running = False
        if self.thread:
            self.thread.join(timeout=2)
```

### 5.3 自適應掃描策略

```python
# adaptive_scanning.py

import numpy as np

class AdaptiveSweep:
    """自適應參數掃描"""
    
    @staticmethod
    def generate_adaptive_points(func, start, stop, 
                                initial_points=10,
                                refinement_factor=0.1):
        """
        根據函數梯度自適應生成掃描點
        
        高梯度區域 → 更多採樣點
        低梯度區域 → 較少採樣點
        """
        
        # 初始均勻採樣
        points = np.linspace(start, stop, initial_points)
        values = [func(p) for p in points]
        
        # 計算梯度
        gradients = np.gradient(values)
        
        # 規範化梯度為採樣密度
        sampling_density = np.abs(gradients) / np.max(np.abs(gradients))
        
        # 在高梯度區域插入額外點
        refined_points = []
        for i in range(len(points) - 1):
            refined_points.append(points[i])
            
            num_insertions = max(1, int(sampling_density[i] * initial_points * refinement_factor))
            
            inserted = np.linspace(points[i], points[i+1], num_insertions + 2)[1:-1]
            refined_points.extend(inserted)
        
        refined_points.append(points[-1])
        
        return np.array(refined_points)
```

---

## 第六部分：進階功能

### 6.1 自動化校準流程

```python
# auto_calibration.py

class AutomaticCalibration:
    """自動儀器校準"""
    
    def __init__(self, controller: QCoDesMarimo):
        self.controller = controller
        self.calibration_data = {}
    
    def calibrate_reference_voltage(self, dac_param, ref_meter,
                                   test_points=[0, 1, 5, 10]):
        """
        校準DAC輸出
        
        流程:
        1. 對每個測試點設置DAC
        2. 用參考表測量實際輸出
        3. 計算校準係數
        """
        
        calibration = {}
        
        for set_val in test_points:
            dac_param.set(set_val)
            time.sleep(0.5)  # 穩定時間
            
            measured = ref_meter.get()
            calibration[set_val] = measured
        
        # 擬合校準曲線
        from numpy.polynomial import polynomial as P
        
        x = np.array(list(calibration.keys()))
        y = np.array(list(calibration.values()))
        
        coeffs = P.polyfit(x, y, 1)  # 線性擬合
        
        self.calibration_data['dac_calibration'] = {
            'coefficients': coeffs,
            'measured_points': calibration
        }
        
        return coeffs
    
    def apply_calibration(self, raw_value, calibration_type):
        """應用校準補正"""
        
        if calibration_type not in self.calibration_data:
            return raw_value
        
        coeffs = self.calibration_data[calibration_type]['coefficients']
        return np.polyval(coeffs[::-1], raw_value)
```

### 6.2 數據導出與報告生成

```python
# export_and_reporting.py

import json
from datetime import datetime
from pathlib import Path

class ExperimentReporter:
    """實驗報告生成"""
    
    def __init__(self, experiment_name, output_dir=None):
        self.experiment_name = experiment_name
        self.output_dir = Path(output_dir or './reports')
        self.output_dir.mkdir(exist_ok=True)
        self.metadata = {
            'name': experiment_name,
            'timestamp': datetime.now().isoformat(),
            'measurements': []
        }
    
    def add_measurement(self, name: str, data: dict, figure=None):
        """添加測量結果"""
        
        self.metadata['measurements'].append({
            'name': name,
            'timestamp': datetime.now().isoformat(),
            'data_points': len(data.get('data_points', []))
        })
        
        # 保存數據為CSV
        df = pd.DataFrame(data.get('data_points', []))
        csv_path = self.output_dir / f"{name}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
        df.to_csv(csv_path, index=False)
        
        # 保存圖表為HTML
        if figure:
            html_path = self.output_dir / f"{name}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.html"
            figure.write_html(html_path)
    
    def generate_report(self) -> str:
        """生成實驗報告"""
        
        report_path = self.output_dir / f"{self.experiment_name}_report.json"
        
        with open(report_path, 'w') as f:
            json.dump(self.metadata, f, indent=2)
        
        return str(report_path)
```

---

## 第七部分：故障排除與最佳實踐

### 7.1 常見問題

| 問題 | 原因 | 解決方案 |
|------|------|--------|
| UI更新無效 | 單元格非反應式 | 使用 `@mo.reactive` 或 `.value` 屬性 |
| 儀器超時 | 通訊延遲 | 增加 `timeout` 參數；檢查連接 |
| 內存洩漏 | 訂閱未清理 | 在 `finally` 塊中調用 `subscriber.stop()` |
| 數據不一致 | 競態條件 | 使用鎖或隊列同步 |
| 圖表無法互動 | Plotly版本不兼容 | 更新至最新版本 |

### 7.2 性能優化建議

```python
# performance_optimization.py

class PerformanceOptimizer:
    
    @staticmethod
    def batch_measurements(param, values, batch_size=50):
        """批量測量以減少通訊開銷"""
        for i in range(0, len(values), batch_size):
            batch = values[i:i+batch_size]
            yield batch
    
    @staticmethod
    def cache_parameter_values(param, cache_duration=60):
        """緩存參數值以減少讀取"""
        import functools
        import time
        
        @functools.lru_cache(maxsize=1)
        def cached_get():
            return param.get()
        
        return cached_get
    
    @staticmethod
    def async_measurements(sweep_param, measured_params, 
                          sweep_values):
        """非同步多參數測量"""
        import asyncio
        from concurrent.futures import ThreadPoolExecutor
        
        async def measure_point(set_val):
            sweep_param.set(set_val)
            results = {}
            
            with ThreadPoolExecutor() as executor:
                for param in measured_params:
                    results[param.name] = await asyncio.get_event_loop().run_in_executor(
                        executor, param.get
                    )
            
            return results
        
        # 並行測量點
        return asyncio.gather(*[measure_point(v) for v in sweep_values])
```

### 7.3 安全與安全性

```python
# security_considerations.py

class SecurityManager:
    """安全管理"""
    
    @staticmethod
    def validate_sweep_parameters(param, start, stop, max_step=1.0):
        """驗證掃描參數以防止危險操作"""
        
        if start < param.vals.min_value or stop > param.vals.max_value:
            raise ValueError("掃描範圍超出允許值")
        
        if abs(stop - start) < max_step:
            raise ValueError("掃描範圍過小")
        
        return True
    
    @staticmethod
    def create_safe_measurement_context(timeout=300):
        """創建安全的測量上下文"""
        
        @contextmanager
        def safe_measurement():
            start_time = time.time()
            
            try:
                yield
            finally:
                elapsed = time.time() - start_time
                if elapsed > timeout:
                    raise TimeoutError(f"測量超時: {elapsed}s > {timeout}s")
        
        return safe_measurement
```

---

## 第八部分：部署與推廣

### 8.1 容器化部署

```bash
# build.sh

#!/bin/bash

# 構建Docker映像
docker build -t marimo-qcodes:latest .

# 運行容器
docker run \
  -p 8080:8080 \
  -v $(pwd)/qcodes_data:/data \
  -v $(pwd)/notebooks:/app/notebooks \
  -e QCODES_DB_PATH=/data/qcodes.db \
  marimo-qcodes:latest

# 推送到Docker Hub
docker tag marimo-qcodes:latest username/marimo-qcodes:latest
docker push username/marimo-qcodes:latest
```

### 8.2 本地開發設置

```bash
# setup.sh

#!/bin/bash

# 創建虛擬環境
python -m venv venv
source venv/bin/activate  # 或 Windows: venv\Scripts\activate

# 安裝依賴
pip install -r requirements.txt
pip install -e .

# 下載示例筆記本
git clone <repo-url> notebooks

# 啟動Marimo服務器
marimo edit notebooks/quantum_dot_experiment.py
```

### 8.3 多實驗室協作

```python
# collaboration.py

class ExperimentRepository:
    """實驗版本控制與共享"""
    
    def __init__(self, repo_path):
        self.repo_path = Path(repo_path)
    
    def save_experiment(self, notebook_path, metadata: dict):
        """保存實驗到版本控制"""
        import subprocess
        
        # 複製筆記本
        backup_dir = self.repo_path / datetime.now().strftime('%Y%m%d_%H%M%S')
        backup_dir.mkdir(parents=True)
        
        shutil.copy(notebook_path, backup_dir / notebook_path.name)
        
        # 保存元數據
        with open(backup_dir / 'metadata.json', 'w') as f:
            json.dump(metadata, f, indent=2)
        
        # Git提交
        subprocess.run(['git', 'add', '.'], cwd=self.repo_path)
        subprocess.run(
            ['git', 'commit', '-m', f"Experiment: {metadata.get('name')}"],
            cwd=self.repo_path
        )
        
        return backup_dir
    
    def load_experiment(self, experiment_id):
        """加載過去的實驗"""
        exp_dir = self.repo_path / experiment_id
        
        if not exp_dir.exists():
            raise FileNotFoundError(f"實驗未找到: {experiment_id}")
        
        with open(exp_dir / 'metadata.json', 'r') as f:
            metadata = json.load(f)
        
        return metadata, exp_dir
```

---

## 附錄A：完整API參考

### Marimo 關鍵方法

- `mo.ui.slider()` - 滑塊輸入
- `mo.ui.dropdown()` - 下拉選單
- `mo.ui.number()` - 數值輸入
- `mo.ui.text()` - 文本輸入
- `mo.ui.checkbox()` - 複選框
- `mo.ui.table()` - 數據表
- `mo.vstack()` / `mo.hstack()` - 佈局
- `mo.md()` - Markdown
- `@mo.reactive` - 反應式單元格

### QCodes 關鍵方法

- `Parameter.set(value)` - 設置參數
- `Parameter.get()` - 讀取參數
- `Measurement.run()` - 執行測量
- `Measurement.register_parameter()` - 註冊參數
- `Measurement.add_subscriber()` - 訂閱數據
- `DataSet.to_dataframe()` - 導出為DataFrame

---

## 附錄B：範例配置文件

```python
# config.py

from dataclasses import dataclass
from pathlib import Path

@dataclass
class ExperimentConfig:
    """實驗配置"""
    
    # 數據存儲
    data_dir: Path = Path.home() / '.qcodes' / 'experiments'
    db_path: Path = Path.home() / '.qcodes' / 'data.db'
    
    # 儀器設置
    instruments: dict = None
    
    # Marimo配置
    marimo_host: str = '0.0.0.0'
    marimo_port: int = 8080
    
    # 測量參數
    sweep_timeout: float = 3600.0  # 1小時
    measurement_write_period: float = 0.5  # 秒
    
    def __post_init__(self):
        self.data_dir.mkdir(parents=True, exist_ok=True)
        
        if self.instruments is None:
            self.instruments = {}

# 使用範例
config = ExperimentConfig(
    instruments={
        'gate': {'address': 'GPIB0::1::INSTR'},
        'bias': {'address': 'GPIB0::2::INSTR'},
        'lockin': {'address': 'GPIB0::10::INSTR'}
    }
)
```

---

## 總結

此整合方案提供：

✅ **統一控制界面** - Marimo UI用於儀器控制  
✅ **自動化測量** - QCodes協議管理複雜掃描  
✅ **實時分析** - 動態數據處理與可視化  
✅ **可重現性** - 完整的元數據與版本控制  
✅ **可擴展性** - 模塊化設計易於定制  
✅ **可部署性** - 容器化與Web應用支持  

該方案特別適用於：
- 量子計算/量子點實驗
- 固態物理測量
- 凝聚態物理研究
- 半導體器件表徵
- 任何需要儀器控制+數據分析的科學實驗

