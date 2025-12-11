# Marimo + QCodes 快速開始指南

## 1. 安裝與環境設置

### 1.1 基本依賴安裝

```bash
# 創建虛擬環境
python -m venv qcodes_marimo_env
source qcodes_marimo_env/bin/activate  # Windows: qcodes_marimo_env\Scripts\activate

# 升級pip
pip install --upgrade pip

# 安裝核心依賴
pip install marimo QCodes numpy pandas scipy matplotlib plotly
```

### 1.2 儀器驅動安裝（按需）

```bash
# GPIB通訊
pip install pyvisa pyvisa-py

# 常見儀器驅動
pip install qcodes-drivers

# 數據可視化
pip install altair jupyter nbformat
```

### 1.3 驗證安裝

```python
# verify_installation.py
import marimo as mo
import qcodes as qc
print(f"Marimo: {mo.__version__}")
print(f"QCodes: {qc.__version__}")
# 應顯示版本信息，無錯誤即成功
```

---

## 2. 第一個應用：簡單掃描

### 2.1 創建基礎筆記本

```python
# first_experiment.py

import marimo as mo
import numpy as np

app = mo.App()

# ============ 第1單元格：導入 ============
import qcodes as qc
from qcodes.instrument_drivers.yokogawa import GS200
import plotly.graph_objects as go

# ============ 第2單元格：儀器初始化 ============

# 模擬儀器（用於測試，不需要真實硬件）
class MockInstrument:
    def __init__(self, name):
        self.name = name
        self.voltage = MockParameter("voltage", 0, -10, 10)
    
class MockParameter:
    def __init__(self, name, default, min_val, max_val):
        self.name = name
        self.value = default
        self.min_value = min_val
        self.max_value = max_val
    
    def set(self, value):
        self.value = np.clip(value, self.min_value, self.max_value)
    
    def get(self):
        return self.value

# 初始化儀器
source = MockInstrument("voltage_source")
measurement_device = MockInstrument("detector")

mo.md("""
# 簡單掃描實驗

此應用演示：
- 使用Marimo UI控制參數
- 執行QCodes風格的掃描
- 實時繪製結果
""")

# ============ 第3單元格：UI控制 ============

mo.vstack([
    mo.md("## 掃描參數設置"),
    
    mo.hstack([
        mo.md("**起始電壓 (V):**"),
        start_voltage := mo.ui.slider(start=-10, stop=10, step=0.5, value=-5)
    ]),
    
    mo.hstack([
        mo.md("**結束電壓 (V):**"),
        stop_voltage := mo.ui.slider(start=-10, stop=10, step=0.5, value=5)
    ]),
    
    mo.hstack([
        mo.md("**掃描點數:**"),
        num_points := mo.ui.number(start=5, stop=200, step=5, value=50)
    ]),
    
    mo.hstack([
        mo.md("**掃描速率 (V/s):**"),
        sweep_rate := mo.ui.slider(start=0.1, stop=10, step=0.1, value=1.0)
    ])
])

# ============ 第4單元格：掃描執行 ============

# 反應式單元格：當掃描參數變化時自動執行
sweep_data = []

for i, voltage in enumerate(np.linspace(start_voltage.value, stop_voltage.value, int(num_points.value))):
    source.voltage.set(voltage)
    
    # 模擬測量：y = x^2 + 噪聲
    measurement = voltage**2 + np.random.normal(0, 0.1)
    
    sweep_data.append({'voltage': voltage, 'signal': measurement})

mo.md(f"### 掃描完成！收集了 {len(sweep_data)} 個數據點")

# ============ 第5單元格：可視化結果 ============

import pandas as pd

# 轉換為DataFrame
df = pd.DataFrame(sweep_data)

# 創建交互式圖表
fig = go.Figure()

fig.add_trace(go.Scatter(
    x=df['voltage'],
    y=df['signal'],
    mode='lines+markers',
    name='Measurement',
    marker=dict(size=6, color='#1f77b4'),
    line=dict(width=2)
))

fig.update_layout(
    title="電壓掃描結果",
    xaxis_title="電壓 (V)",
    yaxis_title="信號強度 (a.u.)",
    hovermode='x unified',
    template='plotly_white',
    height=500
)

mo.vstack([
    mo.md("## 結果圖表"),
    fig,
    mo.md(f"""
    ### 統計信息
    - **最大值**: {df['signal'].max():.4f}
    - **最小值**: {df['signal'].min():.4f}
    - **平均值**: {df['signal'].mean():.4f}
    - **標準差**: {df['signal'].std():.4f}
    """)
])

# ============ 第6單元格：數據導出 ============

mo.md("""
## 數據導出

點擊下方按鈕下載結果：
""")

# 保存數據為CSV
csv_data = df.to_csv(index=False)

mo.ui.download(
    data=csv_data,
    filename="sweep_results.csv",
    label="📥 下載 CSV"
)
```

### 2.2 運行應用

```bash
# 編輯模式（帶實時更新）
marimo edit first_experiment.py

# 作為應用運行
marimo run first_experiment.py

# 作為Web服務部署
marimo run --host 0.0.0.0 --port 8080 first_experiment.py
```

---

## 3. 構建橋接類

### 3.1 基本版本

```python
# qcodes_marimo_simple.py

import marimo as mo
from qcodes import Parameter, Measurement
import numpy as np
import pandas as pd

class QCoDesMarimo:
    """簡單的QCodes-Marimo橋接"""
    
    def __init__(self):
        self.parameters = {}
        self.uis = {}
        self.last_data = None
    
    def register_parameter(self, name: str, 
                          min_val: float, max_val: float, 
                          step: float = None) -> mo.ui.slider:
        """註冊參數並返回UI控件"""
        
        if step is None:
            step = (max_val - min_val) / 100
        
        ui = mo.ui.slider(
            start=min_val,
            stop=max_val,
            step=step,
            label=name
        )
        
        self.uis[name] = ui
        self.parameters[name] = {
            'min': min_val,
            'max': max_val,
            'step': step,
            'current': (min_val + max_val) / 2
        }
        
        return ui
    
    def create_sweep(self, param_name: str, 
                    measured_func, start=None, stop=None, num_points=50):
        """執行簡單掃描"""
        
        if start is None:
            start = self.parameters[param_name]['min']
        if stop is None:
            stop = self.parameters[param_name]['max']
        
        x_values = np.linspace(start, stop, num_points)
        y_values = [measured_func(x) for x in x_values]
        
        self.last_data = pd.DataFrame({
            'x': x_values,
            'y': y_values
        })
        
        return self.last_data

# 使用範例
# ========
controller = QCoDesMarimo()

# UI生成
voltage_ui = controller.register_parameter(
    "Gate Voltage", -2.0, 2.0, step=0.05
)

# 模擬測量函數
def measurement(voltage):
    # 返回一些與電壓相關的值
    return 100 * np.sin(voltage) + np.random.normal(0, 5)

# 執行掃描
data = controller.create_sweep(
    "Gate Voltage",
    measurement,
    start=-2.0,
    stop=2.0,
    num_points=100
)

# 繪製結果
import plotly.express as px
fig = px.scatter(data, x='x', y='y', title="Simple Sweep")
```

### 3.2 集成到筆記本

```python
# integrated_experiment.py

import marimo as mo
import numpy as np
from qcodes_marimo_simple import QCoDesMarimo

app = mo.App()

# 初始化
controller = QCoDesMarimo()

# UI組件
mo.vstack([
    mo.md("# 集成實驗控制"),
    mo.md("## 掃描設置"),
    
    # 動態生成的UI
    voltage_slider := controller.register_parameter(
        "掃描電壓", -5.0, 5.0, step=0.1
    ),
    
    mo.md(f"當前值: {voltage_slider.value:.2f} V"),
    
    # 執行按鈕
    mo.ui.button(label="執行掃描", on_click=lambda: print("掃描開始！"))
])

# 模擬測量（實際應用中連接真實儀器）
def realistic_measurement(voltage):
    """模擬量子點測量"""
    # 量子點通常顯示振盪行為
    conductance = np.abs(np.sin(voltage * np.pi)) 
    noise = np.random.normal(0, 0.01)
    return conductance + noise

# 執行掃描
sweep_result = controller.create_sweep(
    "掃描電壓",
    realistic_measurement,
    start=-5.0,
    stop=5.0,
    num_points=150
)

# 可視化
import plotly.graph_objects as go

fig = go.Figure()
fig.add_trace(go.Scatter(
    x=sweep_result['x'],
    y=sweep_result['y'],
    mode='lines+markers',
    name='Conductance'
))

fig.update_layout(
    title="量子點特性曲線",
    xaxis_title="門電壓 (V)",
    yaxis_title="導電度 (2e²/h)",
    template='plotly_white'
)

mo.vstack([
    mo.md("## 結果"),
    fig,
    mo.md("""
    ### 數據分析
    """),
    mo.ui.table(sweep_result.head(10))
])
```

---

## 4. 連接真實儀器

### 4.1 使用QCodes驅動

```python
# real_hardware_example.py

import marimo as mo
from qcodes import Instrument, Parameter, VisaInstrument
from qcodes_marimo_simple import QCoDesMarimo
import numpy as np

# 定義簡單的儀器驅動
class SimpleVoltageSource(Instrument):
    """簡單電壓源驅動"""
    
    def __init__(self, name, address, **kwargs):
        super().__init__(name, **kwargs)
        
        # 模擬VISA連接
        # 實際應用：self.visa_handle = pyvisa.ResourceManager().open_resource(address)
        self._voltage = 0.0
        
        # 定義參數
        self.add_parameter(
            'voltage',
            get_cmd=self._get_voltage,
            set_cmd=self._set_voltage,
            unit='V',
            label='Output Voltage'
        )
    
    def _set_voltage(self, voltage):
        # 實際：self.visa_handle.write(f"VOLT {voltage}")
        self._voltage = float(voltage)
    
    def _get_voltage(self):
        # 實際：return float(self.visa_handle.query("VOLT?"))
        return self._voltage

# 初始化儀器
# gate_source = SimpleVoltageSource("gate", "GPIB0::1::INSTR")

# 對於演示，使用模擬儀器
gate_source = SimpleVoltageSource("gate", "simulation")

# 在Marimo應用中使用
controller = QCoDesMarimo()

gate_ui = controller.register_parameter(
    "Gate Voltage",
    -10.0,
    10.0,
    step=0.1
)

# 執行掃描（真實儀器）
def measure_signal(voltage):
    gate_source.voltage.set(voltage)  # 設置實際儀器
    # signal = measurement_device.ch1.get()  # 讀取測量值
    return voltage**2  # 模擬測量

data = controller.create_sweep(
    "Gate Voltage",
    measure_signal,
    num_points=100
)

mo.md(f"掃描完成！讀取 {len(data)} 個數據點")
```

### 4.2 多儀器協調

```python
# multi_instrument.py

from qcodes_marimo_simple import QCoDesMarimo
import marimo as mo

class MultiInstrumentController(QCoDesMarimo):
    """多儀器控制器"""
    
    def __init__(self):
        super().__init__()
        self.instruments = {}
    
    def add_instrument(self, name, instrument):
        """添加儀器"""
        self.instruments[name] = instrument
    
    def create_2d_sweep(self, param1_name, param2_name,
                       measured_func, 
                       start1, stop1, num1,
                       start2, stop2, num2):
        """執行2D掃描"""
        
        import numpy as np
        
        x = np.linspace(start1, stop1, num1)
        y = np.linspace(start2, stop2, num2)
        
        data_2d = np.zeros((num1, num2))
        
        for i, x_val in enumerate(x):
            for j, y_val in enumerate(y):
                self.instruments[param1_name].set(x_val)
                self.instruments[param2_name].set(y_val)
                data_2d[i, j] = measured_func()
        
        return x, y, data_2d

# 使用範例
multi_controller = MultiInstrumentController()
# multi_controller.add_instrument("gate", gate_source)
# multi_controller.add_instrument("bias", bias_source)
```

---

## 5. 調試與最佳實踐

### 5.1 常見錯誤

| 錯誤 | 解決方案 |
|------|--------|
| `VISA: No resources found` | 檢查GPIB連接；使用 `pyvisa-py list_resources()` |
| `Parameter not found` | 確保參數名稱正確；檢查儀器驅動 |
| `UI 不更新` | 確保使用 `.value` 屬性；Marimo自動追蹤依賴 |
| `掃描卡住` | 增加超時；檢查儀器通訊延遲 |

### 5.2 性能優化

```python
# 批量測量
def batch_sweep(source, detector, values, batch_size=50):
    results = []
    for i in range(0, len(values), batch_size):
        batch = values[i:i+batch_size]
        for val in batch:
            source.set(val)
            results.append(detector.get())
    return results

# 異步測量（如果支持）
import asyncio

async def async_sweep(source, detector, values):
    results = []
    for val in values:
        source.set(val)
        # 非阻塞等待
        results.append(await asyncio.sleep(0.01))
    return results
```

### 5.3 日誌與診斷

```python
import logging

# 設置日誌
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

qcodes_logger = logging.getLogger('qcodes')
qcodes_logger.setLevel(logging.DEBUG)

# 在掃描中添加日誌
for i, val in enumerate(sweep_values):
    logging.debug(f"Step {i}: Setting {val}")
    source.set(val)
    measurement = detector.get()
    logging.debug(f"Step {i}: Measured {measurement}")
```

---

## 6. 部署清單

- [ ] 安裝所有依賴
- [ ] 驗證儀器連接
- [ ] 測試VISA通訊
- [ ] 創建虛擬測試環境
- [ ] 驗證UI操作
- [ ] 測試數據保存
- [ ] 準備Docker配置
- [ ] 設置數據備份
- [ ] 文檔完整

---

## 7. 後續資源

- 📖 [Marimo 文檔](https://marimo.io)
- 📖 [QCodes 文檔](https://qcodes.github.io/Qcodes/)
- 🔧 [常見儀器驅動](https://github.com/QCoDeS/Qcodes_drivers)
- 💬 [Marimo 社區](https://github.com/marimo-team/marimo)
- 💬 [QCodes 討論](https://github.com/microsoft/Qcodes/discussions)

