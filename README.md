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

# 定义ARIMA模型
def train_arima_model(series, order=(1, 0, 1), test_size=0.2): 
    n_test = int(len(series) * test_size)
    train, test = series[:-n_test], series[-n_test:]

    model = ARIMA(train, order=order)
    result = model.fit()

    forecast = result.forecast(steps=len(test))
    predictions = pd.Series(forecast, index=test.index)

    mae = mean_absolute_error(test, predictions)
    return mae

#计算Global_MAE
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
#SARIMA
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

