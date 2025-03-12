# 预测增长：xxx 种子的数据驱动解决方案 
为保护数据隐私，此项目只展示分析思路

# 项目介绍
本研究旨在帮助xxx种子公司应对预测种子需求和优化包装和采购策略方面的挑战。由于季节性需求波动以及 COVID-19 疫情期间和之后种子需求的意外激增，该公司在确定要维持哪种包装尺寸以及预测单个产品的需求方面面临不确定性。帮助该公司优化库存管理，降低成本。

## 数据结构
旧ERP：2017/07/01 - 2023/06/30 新ERP: 2023/4/1 - 2024/6/30
（每年的历史数据是不同的文件）

| Column Name (旧ERP) | Description            | Column Name（新ERP）|             
|-------------|------------------------|---------------------|
| STATE        |   州           |State
| CUSTNMBR     | 客户ID               |Customer
| SLPRSNID       | 订单途径           |SRID
| SOPNUMBE        | 订单编号               |OrderNbr
| ITEMNMBR     | 产品ID               |InventoryID
| GENUS       | 类别          |CropName
| ITEMDESC        | 细分类别               |Description
| UOFM     | 产品规格              |Productsize
| XTNDPRCE       | 单价          |Amount
| QUANTITY        | 订单数量               |Quantity
| DOCDATE     | 订单日期               |Date
|PRCLEVEL       |客户类别                   |PriceClass

# 1.基础数据处理
```python
过滤数据的必要性：（1）某些 InventoryID 的历史数据有限 （2）时间间隔缺失或不规则
```

旧ERP数据拼接
```python
import pandas as pd

# 定义要合并的文件名列表
filenames = ['2018 Sales Data_Filtered.xlsx', '2019 Sales Data_Filtered.xlsx', '2020 Sales Data_Filtered.xlsx',
             '2021 Sales Data_Filtered.xlsx', '2022 Sales Data_Filtered.xlsx', '2023 Sales Data_Filtered.xlsx',
             '2024 Sales Data GP.xlsx']

# 初始化一个空的列表，用于存储每个文件的数据
dfs = []

# 逐个处理文件，读取并添加到列表中
for file in filenames:
    df = pd.read_excel(file)
    dfs.append(df)

# 将所有的 DataFrame 合并为一个
combined_df = pd.concat(dfs, ignore_index=True)

# 打印合并后的数据
print(combined_df)


# 确保数据已被合并
combined_df['DOCDATE'] = pd.to_datetime(combined_df['DOCDATE'], errors='coerce')  # Convert the Date column to datetime
filtered_df = combined_df[(combined_df['DOCDATE'].dt.year >= 2018) & (combined_df['DOCDATE'].dt.year <= 2024)]
if not filtered_df.empty:
    print("2021-2022 data is present.")
else:
    print("No data found for 2021-2022.")

# 指定日期格式
combined_df['DOCDATE'] = combined_df['DOCDATE'].dt.strftime('%m/%d/%Y')
# 指定列的名称
columns_order = ['State', 'Customer', 'SRID', 'OrderNbr', 'InventoryID', 'CropName', 'Description', 'Productsize',
                 'Amount', 'Quantity', 'Date','PRODUCT CLASS', 'PriceClass']
combined_df.columns = columns_order

```

数据映射
```python

!pip install openpyxl

mapping_inv = pd.read_excel('Mapping Document.xlsx', sheet_name='GP_to_ACU_InvMapping')
mapping_uofm = pd.read_excel('Mapping Document.xlsx', sheet_name='UOFM_mapping')

# 确保 Date 列为 datetime 类型
combined_df['Date'] = pd.to_datetime(combined_df['Date'], errors='coerce')

# 删除 GP_ITEMNMBR 列中所有前导零（使用 str.lstrip 来移除左侧的 '0'）
mapping_inv['GP_ITEMNMBR'] = mapping_inv['GP_ITEMNMBR'].astype(str).str.lstrip('0')

# 标准化
combined_df['InventoryID'] = combined_df['InventoryID'].astype(str).str.strip().str.upper()
combined_df['Productsize'] = combined_df['Productsize'].astype(str).str.strip().str.upper()
mapping_inv['GP_ITEMNMBR'] = mapping_inv['GP_ITEMNMBR'].astype(str).str.strip().str.upper()
mapping_uofm['GP_UOFM'] = mapping_uofm['GP_UOFM'].astype(str).str.strip().str.upper()

# 创建映射字典
inventory_mapping = dict(zip(mapping_inv['GP_ITEMNMBR'], mapping_inv['Acumatica_InventoryCD']))
uofm_mapping = dict(zip(mapping_uofm['GP_UOFM'], mapping_uofm['ACU_UOFM']))

# 替换旧数据库中的列
combined_df['InventoryID'] = combined_df['InventoryID'].map(inventory_mapping)
combined_df['Productsize'] = combined_df['Productsize'].map(uofm_mapping)

# 将 'Description' 和 'CropName' 列的内容转换为小写
combined_df['Description'] = combined_df['Description'].str.lower()
combined_df['CropName'] = combined_df['CropName'].str.lower()
print(combined_df)
```

与新ERP数据拼接
```python
new_erp = pd.read_csv('2024 Sales Data ACU.csv')
columns_order = ['State', 'Customer', 'SRID', 'OrderNbr', 'InventoryID', 'CropName', 'Description', 'Productsize',
                 'Amount', 'Quantity', 'Date','PRODUCT CLASS', 'PriceClass']
new_erp.columns = columns_order

# 数据标准化
new_erp['Description'] = new_erp['Description'].str.lower()
new_erp['CropName'] = new_erp['CropName'].str.lower()

# 确保两个数据集的 Date 列为 datetime 类型，转换为 MM/DD/YYYY 格式
combined_df['Date'] = pd.to_datetime(combined_df['Date'], errors='coerce')
new_erp['Date'] = pd.to_datetime(new_erp['Date'], errors='coerce')
combined_df['Date'] = combined_df['Date'].dt.strftime('%m/%d/%Y')
new_erp['Date'] = new_erp['Date'].dt.strftime('%m/%d/%Y')

# 拼接 old_erp 和 new_erp
combined_erp = pd.concat([combined_df, new_erp], ignore_index=True)
print(combined_erp)

# 输出拼接后的文件
combined_erp.to_csv('combined_sales_data.csv', index=False)

```
数据筛选
```python
# 读取 csv 文件
df = pd.read_csv('combined_sales_data.csv')

# 筛选出 'Productsize' 列中包含 'seed', 'oz', 'lb', 'kg' 的行
keywords = ['seed', 'oz', 'lb', 'kg']
merged_data = df[df['Productsize'].str.contains('|'.join(keywords), case=False, na=False)]
print(merged_data)

#筛选出连续销售7年的种子种类
merged_data['year'] = merged_data['Date'].apply(lambda x: x.year + 1 if x.month >= 7 else x.year)
inventory_grouped = merged_data.groupby(['InventoryID', 'year'])['Quantity'].sum().reset_index()
inventory_year_count = inventory_grouped.groupby('InventoryID')['year'].nunique().reset_index()
valid_inventory = inventory_year_count[inventory_year_count['year'] == 7]['InventoryID']
filtered_inventory = merged_data[merged_data['InventoryID'].isin(valid_inventory)]
```
数据清洗
```python
#观察数据的基本信息
print(filtered_inventory.shape)
print(filtered_inventory.info())


#去除NULL值
filtered_inventory['State'].fillna('Unknown', inplace=True)
print(filtered_inventory.isnull().sum())

# 箱线图查看销售数量的分布
import matplotlib.pyplot as plt
import seaborn as sns
plt.figure(figsize=(10, 5))
sns.boxplot(x=filtered_inventory["Quantity"])
plt.show()

#观察异常值
data_1=filtered_inventory[filtered_inventory['Quantity']>=4000]
data_1.head()

#去除重复数据
duplicates = filtered_inventory[filtered_inventory.duplicated(keep=False)]
duplicates=duplicates.sort_values(by=['InventoryID'])
df = filtered_inventory.drop_duplicates()
print(df.shape)

#处理State列
df['State']=df['State'].str.strip()
df['State']=df['State'].str.upper()
df["State"] = df["State"].replace({"78227":"UNKNOWN","70555":"UNKNOWN","AA":"UNKNOWN","":"UNKNOWN"})
print(df["State"].unique())

#处理CropName列
df['CropName']=df['CropName'].str.strip()
df['CropName'] = df['CropName'].replace(['winter squash', 'summer squash'], 'squash')
df['CropName'] = df['CropName'].replace(['active grow lighting'], 'active ')
df['CropName']=df['CropName'].str.lower()
df['CropName'].value_counts()

#处理Description列
df['Description']=df['Description'].str.strip()
standard_names = {}

#每个inventory ID使用对应的最短的description
for inv_id in df['InventoryID'].unique():
    names = df[df['InventoryID'] == inv_id]['Description']
    standard_name = min(names, key=len)
    standard_names[inv_id] = standard_name
print(standard_names)

def similar_and_shorter(a, b):
        return min(a, b, key=len)
df['Description'] = df.apply(lambda x: similar_and_shorter(x['Description'], standard_names[x['InventoryID']]), axis=1)
print(df)

#去除不合理的负值
negative_sales = df[df["Quantity"] <= 0]
print(negative_sales）
df = df[df["Quantity"] > 0]
invalid_prices = df[df["Amount"] <= 0]
print(invalid_prices)

#转换日期格式
df['Date'] = pd.to_datetime(df['Date'])
#输出文件
df.to_csv('final_cleaned.csv',index=False)

```
# 2.数据透视表转换
```python
#weekly
df['week']=df['Date'].dt.to_period('W').apply(lambda r:r.start_time)
df['week'] = pd.to_datetime(df['week']) 
df.set_index('week', inplace=True)
pivot_table_week = df.pivot_table(values='Quantity', index='week', columns='InventoryID', aggfunc='sum', fill_value=0)

#monthly
df['month']=df['Date'].dt.to_period('M').apply(lambda r:r.start_time)
df['month'] = pd.to_datetime(df['month']) 
df.set_index('month', inplace=True)
pivot_table_month = df.pivot_table(values='Quantity', index='month', columns='InventoryID', aggfunc='sum', fill_value=0)

```


# 3.时间序列模型(实际同时使用了weekly和monthly的数据，由于篇幅限制，只呈现weekly)
#ARIMA
```python

!pip install statsmodels matplotlib

from statsmodels.tsa.arima.model import ARIMA
from sklearn.metrics import mean_absolute_error
from statsmodels.tsa.stattools import adfuller

pivot_table_week["week"] = pd.to_datetime(pivot_table_week["week"], format="%Y-%m-%d")
pivot_table_week.set_index("week", inplace=True)

#检查平稳性
def adf_test(series):
    result = adfuller(series.dropna()) 
    return result[1] 

p_values = pivot_table_week.apply(adf_test, axis=0) 
print("各列的 ADF p-value:\n", p_values)

if (p_values < 0.05).sum() > len(p_values) / 2:
    print("大多数列是平稳的，整体上可能是平稳的")
else:
    print("大多数列是非平稳的，整体上可能是非平稳的")

import matplotlib.pyplot as plt
from statsmodels.graphics.tsaplots import plot_acf, plot_pacf

#绘制ACF/PACF图

plt.figure(figsize=(10, 5))
plot_acf(series, lags=50) 
plt.title('ACF')
plt.show()

plt.figure(figsize=(10, 5))
plot_pacf(series, lags=50) 
plt.title('PACF')
plt.show()

提取Inventory ID
inventory_ids = [col for col in pivot_table_week.columns if col not in ["week", "Quantity"]]
print(inventory_ids)

#ARIMA模型
def train_arima_model(series, order=(1, 0, 1), test_size=0.2): 
    n_test = int(len(series) * test_size)
    train, test = series[:-n_test], series[-n_test:]

    model = ARIMA(train, order=order)
    result = model.fit()

    forecast = result.forecast(steps=len(test))
    predictions = pd.Series(forecast, index=test.index)

    mae = mean_absolute_error(test, predictions)
    return mae

global_absolute_errors = []
for inventory_id in inventory_ids:
    series = pivot_table_week[inventory_id].dropna()

    if len(series) < 104:  
        continue

    mae = train_arima_model(series)

    global_absolute_errors.append(mae)

global_mae = sum(global_absolute_errors) / len(global_absolute_errors)
print(f"Global MAE: {global_mae:.4f}")
```
#SARIMA模型
```Python

import pandas as pd
import matplotlib.pyplot as plt
from statsmodels.tsa.statespace.sarimax import SARIMAX
from sklearn.metrics import mean_absolute_error

def train_sarima_model(series, order=(1, 1, 1), seasonal_order=(1, 1, 1, 12), test_size=0.2):

    n_test = max(1, int(len(series) * test_size))
    train, test = series[:-n_test], series[-n_test:]

    model = SARIMAX(train, order=order, seasonal_order=seasonal_order)
    result = model.fit()

    forecast = result.forecast(steps=len(test))
    predictions = pd.Series(forecast, index=test.index)

    mae = mean_absolute_error(test, predictions)
    return mae


def calculate_global_mae(df, inventory_ids):
    global_absolute_errors = Parallel(n_jobs=-1)(delayed(train_sarima_model)(df[inventory_id].dropna())
                                                 for inventory_id in inventory_ids if len(df[inventory_id].dropna()) >= 104)
    global_mae = sum(global_absolute_errors) / len(global_absolute_errors)
    return global_mae

global_mae = calculate_global_mae(pivot_table_week, inventory_ids)
print(f"SARIMA Global MAE: {global_mae:.4f}")

```
#LSTM模型
```python
class Timer():

	def __init__(self):
		self.start_dt = None

	def start(self):
		self.start_dt = dt.datetime.now()

	def stop(self):
		end_dt = dt.datetime.now()
		print('Time taken: %s' % (end_dt - self.start_dt))

import pandas as pd
import numpy as np
import math
from numpy import newaxis
import datetime as dt

import matplotlib.pyplot as plt
import seaborn as sns
%matplotlib inline

from tensorflow.keras.layers import Dense, Activation, Dropout, LSTM，Bidirectional
from tensorflow.keras.models import Sequential, load_model
from sklearn.metrics import mean_squared_error
import matplotlib.pyplot as plt
from sklearn.preprocessing import MinMaxScaler
from sklearn.preprocessing import StandardScaler
from keras.callbacks import EarlyStopping, ModelCheckpoint

class SalesDataLoader():

    def __init__(self, dataframe, split, cols):
        i_split = int(len(dataframe) * split)
        self.data_train = dataframe[cols].values[:i_split]
        self.data_test  = dataframe[cols].values[i_split:]
        self.len_train  = len(self.data_train)
        self.len_test   = len(self.data_test)
        self.len_train_windows = None

    def get_test_data(self, seq_len, normalise=False):

        data_windows = []
        for i in range(self.len_test - seq_len):
            data_windows.append(self.data_test[i:i+seq_len])

        data_windows = np.array(data_windows).astype(float)
        data_windows = self.normalise_windows(data_windows, single_window=False) if normalise else data_windows

        x = data_windows[:, :-1]
        y = data_windows[:, -1, [0]]
        return x, y

    def get_train_data(self, seq_len, normalise=False):
        
        data_x = []
        data_y = []
        for i in range(self.len_train - seq_len):
            x, y = self._next_window(i, seq_len, normalise)
            data_x.append(x)
            data_y.append(y)
        return np.array(data_x), np.array(data_y)

    def _next_window(self, i, seq_len, normalise):
        
        window = self.data_train[i:i+seq_len]
        window = self.normalise_windows(window, single_window=True)[0] if normalise else window
        x = window[:-1]
        y = window[-1, [0]]
        return x, y

    def normalise_windows(self, window_data, single_window=False):
        
        normalised_data = []
        window_data = [window_data] if single_window else window_data
        for window in window_data:
            normalised_window = []
            for col_i in range(window.shape[1]):
                normalised_col = [((float(p) / float(window[0, col_i])) - 1) if window[0, col_i] != 0 else 0 for p in window[:, col_i]]
                normalised_window.append(normalised_col)
            normalised_window = np.array(normalised_window).T
            normalised_data.append(normalised_window)
        return np.array(normalised_data)

seq_len = 5
batch_size = 32
cols = pivot_table_week.columns
loader = SalesDataLoader(pivot_table_week, split=0.8, cols=cols)

#Create training and test data
x_train, y_train = loader.get_train_data(seq_len, normalise=True)
x_test, y_test = loader.get_test_data(seq_len, normalise=True)

print(x_train.shape)
print(y_train.shape)
print(x_test.shape)
print(y_test.shape)

df_iv1 = pivot_table_week.iloc[:,0]
df1 = df_iv1.to_frame(name = 'Quantity')
df1.head()

plt.figure(figsize=(10, 6))
plt.step(df.index, df1['Quantity'], where='mid', linestyle='-', marker='o', color='b', label='Quantity')
plt.title('Discrete Unit Changes Over Time')
plt.xlabel('Week')
plt.ylabel('Quantity')
plt.grid(True)
plt.xticks(rotation=45)  # Rotate the x-axis labels for better readability
plt.tight_layout()

plt.show()

split_ratio = 0.8
cols = ['00009STDCRU-1KSD']

data_loader = SalesDataLoader(df, split=split_ratio, cols=cols)

seq_len = 4
x_train, y_train = data_loader.get_train_data(seq_len, normalise=True)
x_test, y_test = data_loader.get_test_data(seq_len, normalise=True)

print(f"x_train shape: {x_train.shape}, y_train shape: {y_train.shape}")
print(f"x_test shape: {x_test.shape}, y_test shape: {y_test.shape}")

class Model():
    
    def __init__(self):
        self.model = Sequential()

    def load_model(self, filepath):
        print(f'[Model] Loading model from file {filepath}')
        self.model = load_model(filepath)

    def build_model(self, configs):
        timer = Timer()
        timer.start()

        for layer in configs['model']['layers']:
            neurons = layer.get('neurons')
            dropout_rate = layer.get('rate')
            activation = layer.get('activation')
            return_seq = layer.get('return_seq')
            input_timesteps = layer.get('input_timesteps')
            input_dim = layer.get('input_dim')

            if layer['type'] == 'dense':
                self.model.add(Dense(neurons, activation=activation))
            if layer['type'] == 'lstm':
                self.model.add(LSTM(neurons, input_shape=(input_timesteps, input_dim), return_sequences=return_seq))
                
            if layer['type'] == 'dropout':
                self.model.add(Dropout(dropout_rate))

        self.model.compile(loss=configs['model']['loss'], 
                   optimizer=configs['model']['optimizer'], 
                   metrics=configs['model']['metrics'])

        print('[Model] Model Compiled')
        timer.stop()

    def train(self, x, y, epochs, batch_size):
        timer = Timer()
        timer.start()
        print('[Model] Training Started')
        print(f'[Model] {epochs} epochs, {batch_size} batch size')

        # Save model in the current directory
        save_fname = f'{dt.datetime.now().strftime("%d%m%Y-%H%M%S")}-e{epochs}.keras'
        callbacks = [
            EarlyStopping(monitor='val_loss', patience=5),
            ModelCheckpoint(filepath=save_fname, monitor='val_loss', save_best_only=True)
        ]
        
        # Train the model and display the progress
        self.model.fit(
            x,
            y,
            epochs=epochs,
            batch_size=batch_size,
            validation_split=0.2,  # Use 20% of training data for validation
            callbacks=callbacks
        )
        self.model.save(save_fname)

        print(f'[Model] Training Completed. Model saved as {save_fname}')
        timer.stop()

    def predict_point_by_point(self, data):
       
        print('[Model] Predicting Point-by-Point...')
        predicted = self.model.predict(data)
        predicted = np.reshape(predicted, (predicted.size,))
        return predicted

    def predict_sequences_multiple(self, data, window_size, prediction_len):
       
        print('[Model] Predicting Sequences Multiple...')
        prediction_seqs = []
        for i in range(int(len(data) / prediction_len)):
            curr_frame = data[i * prediction_len]
            predicted = []
            for j in range(prediction_len):
                predicted.append(self.model.predict(curr_frame[newaxis, :, :])[0, 0])
                curr_frame = curr_frame[1:]
                curr_frame = np.insert(curr_frame, [window_size - 2], predicted[-1], axis=0)
            prediction_seqs.append(predicted)
        return prediction_seqs

    def predict_sequence_full(self, data, window_size):
        
        print('[Model] Predicting Sequences Full...')
        curr_frame = data[0]
        predicted = []
        for i in range(len(data)):
            predicted.append(self.model.predict(curr_frame[newaxis, :, :])[0, 0])
            curr_frame = curr_frame[1:]
            curr_frame = np.insert(curr_frame, [window_size - 2], predicted[-1], axis=0)
        return predicted

configs = {
    'model': {
        'loss': 'mean_squared_error', 
        'optimizer': 'adam',
        'metrics': ['mae'], 
        'layers': [
            {
                'type': 'lstm',
                'neurons': 50,
                'input_timesteps': 10,
                'input_dim': 3,
                'return_seq': True
            },
            {
                'type': 'dropout',
                'rate': 0.2
            },
            {
                'type': 'lstm',
                'neurons': 100,
                'return_seq': False
            },
            {
                'type': 'dense',
                'neurons': 1,
                'activation': 'linear'
            }
        ]
    }
}


model = Model()
model.build_model(configs)

epochs = 50
batch_size = 32
model.train(x_train, y_train, epochs=epochs, batch_size=batch_size)

predicted_values = model.predict_point_by_point(x_test)

print(f"Predicted values shape: {predicted_values.shape}")
print(f"Actual values shape: {y_test.shape}")

y_test_flat = y_test.flatten()

assert len(predicted_values) == len(y_test_flat), 

plt.figure(figsize=(12, 6))
plt.plot(y_test_flat, label='Actual Values', color='blue')
plt.plot(predicted_values, label='Predicted Values', color='red')
plt.title('Actual vs Predicted Values on Test Data')
plt.xlabel('Time Steps')
plt.ylabel('Values')
plt.legend()
plt.show()

test_loss, test_mae = model.model.evaluate(x_test, y_test, verbose=1)

print(f"Test Loss (MSE): {test_loss}")
print(f"Test MAE: {test_mae}"）、

window_size = 3

predicted_values = model.predict_sequence_full(x_test, window_size)

plt.plot(y_test.flatten(), label='Actual Values', color='blue')
plt.plot(predicted_values, label='Predicted Values', color='red')
plt.title('Full Sequence Prediction vs Actual (Test Set)')
plt.xlabel('Time Steps')
plt.ylabel('Values')
plt.legend()
plt.show()

y_true = y_test.flatten()  # Flatten if necessary, so it's 1D
y_pred = np.array(predicted_values)  # Predicted values should already be 1D

numerator = np.sum(np.abs(y_true - y_pred))

mean_y_true = np.mean(y_true)
denominator = np.sum(np.abs(y_true - mean_y_true))

rae = numerator / denominator
print(f"Relative Absolute Error (RAE): {rae}")
```
#XGboost


```python
!pip install pandas-datareader holidays
import pandas_datareader.data as web
import datetime
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from xgboost import XGBRegressor
from sklearn.metrics import mean_squared_error, mean_absolute_error
import numpy as np
from tqdm import tqdm
from sklearn.model_selection import TimeSeriesSplit
from copy import copy, deepcopy
import holidays
np.random.seed(42)

#设置滞后期
look_back = 30 

def create_features(df):
    for i in range(1, look_back + 1):
        df[f'lag_{i}'] = df['quantity'].shift(i)
    df['day_of_month'] = df.index.day
    df['month'] = df.index.month
    df['rolling_mean_7'] = df['quantity'].shift(1).rolling(window=7).mean()
    df[f'rolling_mean_{look_back}'] = df['quantity'].shift(1).rolling(window=look_back).mean()

    us_holidays = holidays.country_holidays('US')
    df['is_holidays'] = df.index.to_series().apply(lambda x: 1 if x in us_holidays else 0)

    covid_start = pd.to_datetime('2020-03-01')
    covid_end = pd.to_datetime('2021-06-30')
    df['is_covid'] = df.index.to_series().apply(lambda x: 1 if covid_start <= x <= covid_end else 0)
    return df

train_index = np.array(list(range(0, int(len(df)*0.8))))
test_index = np.array(list(range(int(len(df)*0.8), len(df))))
# Test 和 Train 以分别进行了数据基本处理和清洗，此处不再赘述

#选择3个ID进行训练
inventory_ids = ['00009STDCRU-10KSD', "00105RAWUTR-1KSD", "02091RAWUTR-500SD"]

for inventory_id in tqdm(inventory_ids):
    print(f"Cross-validation for Inventory ID: {inventory_id}")

    data = ps_data[inventory_id].dropna().values

    original_df = pd.DataFrame(data, columns=['quantity'], index=ps_data.index)
    df = create_features(deepcopy(original_df))

    X = df.drop('quantity', axis=1).values.tolist()
    Y = df["quantity"].values.tolist()

    trainX = [X[i] for i in train_index if i >= look_back]
    trainY = [Y[i] for i in train_index if i >= look_back]


    model = XGBRegressor(n_estimators=25, learning_rate=0.1, random_state=42)
    model.fit(trainX, trainY)

    y_pred_train = model.predict(trainX)

    history = deepcopy(df)
    for i, x in enumerate(test_index):
        history.loc[history.index[i], "quantity"] = 0

    testY = [Y[i] for i in test_index]
    y_pred_test = []
    for test_id in test_index:
        test_subset = create_features(history[["quantity"]].iloc[test_id - look_back:test_id + 1])
        testX = test_subset.iloc[-1].drop("quantity").tolist()
        y_pred = model.predict(np.array(testX).reshape((1, -1)))[0]
        history.loc[history.index[test_id], "quantity"] = y_pred
        y_pred_test.append(y_pred)

    y_pred_train = np.maximum(y_pred_train, 0)
    y_pred_test = np.maximum(y_pred_test, 0)

    train_mae = mean_absolute_error(trainY, y_pred_train)
    test_mae = mean_absolute_error(testY, y_pred_test)

    print(f"Train MAE: {train_mae:.3f}, Test MAE: {test_mae:.3f}")

    plt.figure(figsize=(10, 6))

    plt.plot(range(len(trainY)), trainY, label='Train Actual')

    plt.plot(range(len(trainY), len(trainY) + len(testY)), testY, color='red', label='Test Actual')

    plt.plot(range(len(trainY), len(trainY) + len(testY)), y_pred_test, color='blue', label='Test Predicted')

    plt.title(f"XGBoost Predictions for {inventory_id}, Train MAE: {train_mae:.3f}, Test MAE: {test_mae:.3f}")
    plt.ylabel('Quantity')
    plt.xlabel('Time Steps')
    plt.legend()
    plt.show()
```

#Prophet
```python
!pip install prophet
from prophet import Prophet
from sklearn.metrics import mean_absolute_percentage_error as mape
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt


# COVID-19
covid_periods = pd.DataFrame({
    'holiday': 'COVID-19',
    'ds': pd.to_datetime(['2020-03-01', '2020-04-01', '2020-05-01',
                          '2021-01-01', '2021-02-01', '2021-03-01']),
    'lower_window': 0,
    'upper_window': 30
})

holidays = pd.concat([covid_periods])

covid_periods = pd.DataFrame({
    'holiday': 'COVID-19',
    'ds': pd.to_datetime(['2020-01-01', '2020-02-01', '2020-03-01', '2020-04-01', '2020-05-01',
                          '2020-06-01', '2020-07-01', '2020-08-01', '2020-09-01', '2020-10-01',
                          '2020-11-01', '2020-12-01', '2021-01-01', '2021-02-01', '2021-03-01',
                          '2021-04-01', '2021-05-01', '2021-06-01', '2021-07-01', '2021-08-01',
                          '2021-09-01', '2021-10-01', '2021-11-01', '2021-12-01',
                          '2022-01-01', '2022-02-01', '2022-03-01', '2022-04-01', '2022-05-01',
                          '2022-06-01', '2022-07-01', '2022-08-01', '2022-09-01', '2022-10-01',
                          '2022-11-01', '2022-12-01']),
    'lower_window': 0,
    'upper_window': 30  # 30-day window to account for monthly impact
})

# read csv
data = pd.read_csv('pivot_table_week.csv')
weekly_sales = data.groupby([pd.Grouper(key='Dates', freq='W'), 'InventoryID'])['Quantity'].sum().reset_index()
print(weekly_sales.head())

def split_data(data, train_size=0.8, random_seed=42):
    np.random.seed(random_seed)

    data = data.sort_values(by='Dates')

    total_len = len(data)
    train_end = int(total_len * train_size)

    train_data = data.iloc[:train_end]      
    test_data = data.iloc[train_end:]       

    return train_data, test_data

train_data, test_data = split_data(weekly_sales)

def train_test_prophet_wk(inventory_id, train_data, test_data, periods=12, freq='W'):
    train_data = train_data[train_data['InventoryID'] == inventory_id]
    test_data = test_data[test_data['InventoryID'] == inventory_id]

    train_data = train_data[['Dates', 'Quantity']].rename(columns={'Dates': 'ds', 'Quantity': 'y'})
    test_data = test_data[['Dates', 'Quantity']].rename(columns={'Dates': 'ds', 'Quantity': 'y'})

    model = Prophet(yearly_seasonality=True, weekly_seasonality=True, daily_seasonality=False, holidays=covid_periods)

    model.fit(train_data)

    test_future = test_data[['ds']]
    test_forecast = model.predict(test_future)

    test_mae = mean_absolute_error(test_data['y'], test_forecast['yhat'])
    print(f"Test MAE: {test_mae:.2f}")

    future_dates = model.make_future_dataframe(periods=periods, freq=freq)
    forecast = model.predict(future_dates)

    print(f"Future forecast for {inventory_id}:")
    print(forecast[['ds', 'yhat', 'yhat_lower', 'yhat_upper']].tail(periods))

    model.plot(forecast)
    plt.title(f"Forecast for InventoryID: {inventory_id}")
    plt.show()

    return forecast

import numpy as np
from sklearn.metrics import mean_absolute_error

def train_test_prophet_wmae(inventory_id, train_data, test_data, periods=12, freq='W'):

    train_data = train_data[train_data['InventoryID'] == inventory_id]
    test_data = test_data[test_data['InventoryID'] == inventory_id]

    train_data = train_data[['Dates', 'Quantity']].rename(columns={'Dates': 'ds', 'Quantity': 'y'})
    test_data = test_data[['Dates', 'Quantity']].rename(columns={'Dates': 'ds', 'Quantity': 'y'})

    model = Prophet(yearly_seasonality=True, weekly_seasonality=False, daily_seasonality=False, holidays=holidays)
  
    model.fit(train_data)
    
    test_future = test_data[['ds']]
    test_forecast = model.predict(test_future)

    test_forecast = test_forecast[['ds', 'yhat']]
    merged_data = pd.merge(test_data, test_forecast, on='ds')

    absolute_errors = np.abs(merged_data['y'] - merged_data['yhat'])

    total_sales = merged_data['y'].sum()

    min_sales_threshold = 1.0
    sales_with_threshold = np.maximum(merged_data['y'], min_sales_threshold)

    weights = sales_with_threshold / sales_with_threshold.sum()
    test_wmae = np.average(absolute_errors, weights=weights)
    print(f"Weighted Test MAE: {test_wmae:.2f}")

    future_dates = model.make_future_dataframe(periods=periods, freq=freq)
    forecast = model.predict(future_dates)

    print(f"Future forecast for {inventory_id}:")
    print(forecast[['ds', 'yhat', 'yhat_lower', 'yhat_upper']].tail(periods))

    model.plot(forecast)
    plt.title(f"Forecast for InventoryID: {inventory_id}")
    plt.show()

    return forecast

#Test
inventory_id_example = '02091RAWUTR-500SD'
forecast = train_test_prophet_wmae(inventory_id_example, train_data, test_data)

def global_mae_prophet(train_data, test_data, inventory_ids, periods=12, freq='W'):
   
    total_error = 0
    total_count = 0

    for inventory_id in inventory_ids:
        train_subset = train_data[train_data['InventoryID'] == inventory_id]
        test_subset = test_data[test_data['InventoryID'] == inventory_id]

        if train_subset.empty or test_subset.empty:
            continue

        train_subset = train_subset[['Dates', 'Quantity']].rename(columns={'Dates': 'ds', 'Quantity': 'y'})
        test_subset = test_subset[['Dates', 'Quantity']].rename(columns={'Dates': 'ds', 'Quantity': 'y'})

        model = Prophet(yearly_seasonality=True, weekly_seasonality=False, daily_seasonality=False, holidays=holidays)
        model.fit(train_subset)

        test_future = test_subset[['ds']]
        test_forecast = model.predict(test_future)
        test_forecast = test_forecast[['ds', 'yhat']]
        merged_data = pd.merge(test_subset, test_forecast, on='ds')

        absolute_errors = np.abs(merged_data['y'] - merged_data['yhat'])

        total_error += absolute_errors.sum()
        total_count += len(absolute_errors)

    global_mae = total_error / total_count if total_count > 0 else np.nan
    return global_mae

inventory_ids = test_data['InventoryID'].unique()
global_mae = global_mae_prophet(train_data, test_data, inventory_ids)
print("Global MAE:", global_mae)
```
# 4.不同种子之间销量的关系
#大类

```python
pip install pingouin

import numpy as np
import pingouin as pg

df = pd.read_csv("final_cleaned.csv")
df = df.drop(columns=['State','Customer','SRIDs','OrderNbr','Descriptions','PRODUCTSIZE','Amount','PRODUCT_CLASS','PriceClass','InventoryID'])
quantity_wide = df.pivot_table(index='Dates', columns='CropName', values='Quantity', aggfunc='sum', fill_value=0)
print(quantity_wide)

correlation_matrix = quantity_wide.corr()
print(correlation_matrix)

correlation_results = pg.pairwise_corr(quantity_wide, method='pearson')
print(correlation_results)
correlation_results.to_excel('p-value.xlsx', index=False)  
```

#小类
```python
quantity_wide_sub = df.pivot_table(index='Dates', columns='Description', values='Quantity', aggfunc='sum', fill_value=0)
print(quantity_wide_sub)
correlation_matrix_sub = quantity_wide_sub.corr()
print(correlation_matrix_sub)

correlation_results1 = pg.pairwise_corr(quantity_wide_sub, method='pearson')
print(correlation_results1)
correlation_results1.to_excel('p-value1.xlsx', index=False)  
```
#无法确定高相关度是种子类别之间的影响还是与季节有关

```python
!pip install mlxtend
import mlxtend
import pandas as pd
from mlxtend.frequent_patterns import apriori, association_rules
df = pd.read_csv("final_cleaned.csv")
df=df.drop(columns=['State','SRIDs','InventoryID','Descriptions','PRODUCTSIZE','Amount','PRODUCT_CLASS','PriceClass'])
df_2 = df.sort_values(by=['Customer','OrderNbr'])
print(df_2.head(10))
basket = df.groupby(['Customer', 'CropName'])['Quantity'].sum().unstack().fillna(0)
basket.head()
basket = basket.applymap(lambda x: 1 if x > 0 else 0)
frequent_itemsets = apriori(basket, min_support=0.01, use_colnames=True)
print(frequent_itemsets.head(100))
rules = association_rules(frequent_itemsets, metric="lift", min_threshold=1, num_itemsets=len(frequent_itemsets))
print(rules[['antecedents', 'consequents', 'support', 'confidence', 'lift']])
rules.to_excel('customer-relation.xlsx', index=False)

```

#VAR model  捕获多个变量之间的相互依赖关系
```python
from statsmodels.tsa.api import VAR
from sklearn.metrics import mean_absolute_error
import numpy as np

data = pd.read_csv('final_cleaned')
data['Dates'] = pd.to_datetime(data['Dates'], format='%m/%d/%Y')
monthly_sales = data.groupby([pd.Grouper(key='Dates', freq='M'),'CropName','InventoryID'])['Quantity'].sum().reset_index()
print(monthly_sales.head())

print("Unique CropNames:", monthly_sales['CropName'].unique())
data_filtered = monthly_sales[(monthly_sales['CropName'].str.strip().str.lower() == 'pumpkin') |
                              (monthly_sales['CropName'].str.strip().str.lower() == 'squash')]
print("Earliest date after filtering:", data_filtered['Dates'].min())
print("Latest date after filtering:", data_filtered['Dates'].max())

print(data_filtered.head())

pivoted_data = data_filtered.pivot_table(index='Dates', columns=['CropName', 'InventoryID'], values='Quantity').fillna(0)
print(pivoted_data.head())

#ADF test
from statsmodels.tsa.stattools import adfuller
from statsmodels.tsa.api import VAR
from sklearn.metrics import mean_absolute_error

def adf_test(series, title=''):
   
    series = series.dropna()  # remove NaN
    print(f'Augmented Dickey-Fuller Test: {title}')
    result = adfuller(series, autolag='AIC')
    labels = ['ADF Test Statistic', 'p-value', '# Lags Used', '# Observations Used']
    for value, label in zip(result, labels):
        print(f'{label} : {value}')
    if result[1] <= 0.05:
        print("Result: The series is stationary") # return whether the data is stationary or not
        return True
    else:
        print("Result: The series is not stationary")
        return False

differenced_data = pivoted_data.copy()
initial_values = {}

for column in pivoted_data.columns:
    if not adf_test(pivoted_data[column], title=column):

        initial_values[column] = pivoted_data[column].iloc[0]

        pivoted_data[column] = pivoted_data[column].diff().dropna()
    else:

        initial_values[column] = pivoted_data[column].iloc[0]

def split_data(data, train_size=0.8, random_seed=42):
    np.random.seed(random_seed)  

    data = data.dropna()  

    data = data.sort_values(by='Dates')

    total_len = len(data)
    train_end = int(total_len * train_size)

    train_data = data.iloc[:train_end]
    test_data = data.iloc[train_end:]

    return train_data, test_data

train_data, test_data = split_data(pivoted_data)

#model
model = VAR(train_data)
model_fitted = model.fit()

lag_order = model_fitted.k_ar
forecast_input = train_data.values[-lag_order:]
forecasted_diff = model_fitted.forecast(y=forecast_input, steps=len(test_data))
forecasted_diff_df = pd.DataFrame(forecasted_diff, index=test_data.index, columns=differenced_data.columns)

forecasted_df = forecasted_diff_df.copy()
for column in forecasted_df.columns:

    if column in initial_values:
        forecasted_df[column] = forecasted_df[column].cumsum() + initial_values[column]
    else:
        print(f"Warning: Initial value for {column} not found in initial_values. Using first forecasted value.")
        first_forecast_value = forecasted_df[column].iloc[0]
        forecasted_df[column] = forecasted_df[column].cumsum() + forecasted_df[column].iloc[0]

#计算MAE
train_size_index = len(train_data)

test_data = test_data.dropna() 
forecasted_df = forecasted_df.loc[test_data.index]

mae_results = {}
for col in test_data.columns:
    actual_len = len(test_data[col])
    forecasted_len = len(forecasted_df[col])

    min_len = min(actual_len, forecasted_len)
    aligned_actual = test_data[col].iloc[:min_len].dropna()
    aligned_forecasted = forecasted_df[col].iloc[:min_len].dropna()

    if len(aligned_actual) == len(aligned_forecasted):
        mae_results[col] = mean_absolute_error(aligned_actual, aligned_forecasted)
    else:
        print(f"Warning: Column {col} has inconsistent lengths after dropping NaNs.")
print("MAE Results:")
for item, mae in mae_results.items():
    print(f"{item}: {mae}")

#计算WMAE
negative_quantities = data[data['Quantity'] < 0]
print("Negative quantities in the original dataset:")
print(negative_quantities)

total_quantity = monthly_sales.groupby('InventoryID')['Quantity'].sum()
weights = total_quantity / total_quantity.sum()

def weighted_mae(y_true, y_pred, weights, min_weight=0.001):

    weights = weights.clip(lower=min_weight)
    mask = ~np.isnan(y_true) & ~np.isnan(y_pred) & ~np.isnan(weights)
    y_true, y_pred, weights = y_true[mask], y_pred[mask], weights[mask]

    if weights.sum() == 0:
        return np.nan

    return (weights * abs(y_true - y_pred)).sum() / weights.sum()

forecasted_df = forecasted_df.loc[test_data.index]
test_data = test_data.dropna()  
forecasted_df = forecasted_df.fillna(method='ffill').fillna(method='bfill')

wmae_results = {}
for col in test_data.columns:
    aligned_actual = test_data[col].dropna()
    aligned_forecasted = forecasted_df[col].iloc[:len(aligned_actual)].dropna()

    if len(aligned_actual) == len(aligned_forecasted):
        weights = aligned_actual
        wmae_results[col] = weighted_mae(aligned_actual, aligned_forecasted, weights)
    else:
        print(f"Warning: Column {col} has inconsistent lengths after alignment.")

print("WMAE Results:")
for item, wmae in wmae_results.items():
    print(f"{item}: {wmae}")

def global_weighted_mae(y_true_df, y_pred_df, weights, min_weight=0.001):
    inventory_ids = y_true_df.columns
    total_error = 0
    total_weight = 0

    for col in inventory_ids:
        y_true = y_true_df[col]
        y_pred = y_pred_df[col]

        weight = weights[col] if col in weights else 0
        if weight < min_weight:
            weight = min_weight

        mask = ~np.isnan(y_true) & ~np.isnan(y_pred)
        y_true, y_pred = y_true[mask], y_pred[mask]  # remove NAN
        mae = mean_absolute_error(y_true, y_pred)

        total_error += mae * weight
        total_weight += weight

    global_wmae = total_error / total_weight if total_weight != 0 else np.nan
    return global_wmae

global_wmae = global_weighted_mae(test_data, forecasted_df, weights=weights)
print("Global WMAE:", global_wmae)

accuracy_percentage = (global_wmae / (143.9623/84))  * 100
print("Model Accuracy Percentage:", accuracy_percentage, "%")

```
# 5.结果评估
```python
#使用GMAE的原因：
#GMAE 会根据不同的商品规模进行调整，确保对不同销售量的商品进行公平比较。它强调高销量商品，从而更全面地了解模型性能。
```

# 6.用户友好界面
```python
#为了简化时间序列模型结果的使用，我们专门为包装部门开发了一个用户友好的界面。
#此界面允许团队成员生成预测，而无需高级数据分析知识。
#用户只需输入关键详细信息，例如作物名称、描述、产品尺寸、开始月份和结束月份，系统就会直接提供相应的预测。
#该工具使包装部门能够高效、独立地做出数据驱动的决策。
```

# 7.建议
```python
#模型建议：
#在查看了所有模型的 Global Mae 后，我们建议xxx公司使用 XGboost 作为其预测模型。
#当他们拥有更多销售数据时，将其更新到模型中以实现更精确的预测。

#包装建议：
#根据预测结果，给予了包装建议，淘汰了113个销量过低的SKU，以期减少成本。
#审查低需求商品：对于销量一直较低的商品，评估它们是否仍能满足客户需求。此审查应在减少库存和探索新产品机会之间取得平衡。
#适应高销量商品：即使某些商品在疫情后销量下降，但它们仍然是高销量产品。使用灵活的采购策略，能够根据市场条件的变化进行调整。
```

## 季节性趋势
![image](https://github.com/user-attachments/assets/41fb02cd-5f60-4356-8a1c-e8a3e41a79f9)
## GMAE
![image](https://github.com/user-attachments/assets/0afc89ec-f8ce-447d-9d55-9a8a59939908)
## 用户友好界面
![image](https://github.com/user-attachments/assets/427a7a71-49c2-490c-8878-907db0be7d2b)





