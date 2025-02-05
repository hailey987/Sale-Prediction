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
| CUSTNMBR     | 客户编号               |Customer
| SLPRSNID       | 订单途径           |SRID
| SOPNUMBE        | 订单编号               |OrderNbr
| ITEMNMBR     | 产品编号               |InventoryID
| GENUS       | 类别          |CropName
| ITEMDESC        | 细分类别               |Description
| UOFM     | 产品规格              |Productsize
| XTNDPRCE       | 单价          |Amount
| QUANTITY        | 订单数量               |Quantity
| DOCDATE     | 订单日期               |Date
|PRCLEVEL       |客户分类                   |PriceClass

# 1.基础数据处理
旧ERP数据拼接
```python
import pandas as pd

# 定义要合并的文件名列表
filenames = ['2018 Sales Data.xlsx', '2019 Sales Data.xlsx', '2020 Sales Data.xlsx',
             '2021 Sales Data.xlsx', '2022 Sales Data.xlsx', '2023 Sales Data.xlsx',
             '2024 Sales Data.xlsx']

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

#指定列的顺序
columns_order = ['State', 'Customer', 'SRID', 'OrderNbr', 'InventoryID', 'CropName', 'Description',
                 'Productsize', 'Amount', 'Quantity', 'Date', 'PriceClass']
combined_df.columns = columns_order

# 确保数据已被合并
combined_df['Date'] = pd.to_datetime(combined_df['Date'], errors='coerce')  # Convert the Date column to datetime
filtered_df = combined_df[(combined_df['Date'].dt.year >= 2018) & (combined_df['Date'].dt.year <= 2024)]
if not filtered_df.empty:
    print("2021-2022 data is present.")
else:
    print("No data found for 2021-2022.")

# 指定日期格式
combined_df['Date'] = combined_df['Date'].dt.strftime('%m/%d/%Y')

# 输出为新CSV文件
combined_df.to_csv('old_erp.csv', index=False)

```

数据映射
```python
!pip install openpyxl

# 加载所需文件
old_data = pd.read_csv('old_erp.csv')
mapping_inv = pd.read_excel('Mapping Document.xlsx', sheet_name='GP_to_ACU_InvMapping')
mapping_uofm = pd.read_excel('Mapping Document.xlsx', sheet_name='UOFM_mapping')

# 确保 Date 列为 datetime 类型
old_data['Date'] = pd.to_datetime(old_data['Date'], errors='coerce')

# 删除 GP_ITEMNMBR 列中所有前导零（使用 str.lstrip 来移除左侧的 '0'）
mapping_inv['GP_ITEMNMBR'] = mapping_inv['GP_ITEMNMBR'].astype(str).str.lstrip('0')

# 标准化
old_data['InventoryID'] = old_data['InventoryID'].astype(str).str.strip().str.upper()
old_data['Productsize'] = old_data['Productsize'].astype(str).str.strip().str.upper()
mapping_inv['GP_ITEMNMBR'] = mapping_inv['GP_ITEMNMBR'].astype(str).str.strip().str.upper()
mapping_uofm['GP_UOFM'] = mapping_uofm['GP_UOFM'].astype(str).str.strip().str.upper()

# 创建映射字典
inventory_mapping = dict(zip(mapping_inv['GP_ITEMNMBR'], mapping_inv['Acumatica_InventoryCD']))
uofm_mapping = dict(zip(mapping_uofm['GP_UOFM'], mapping_uofm['ACU_UOFM']))

# 替换旧数据库中的列
old_data['InventoryID'] = old_data['InventoryID'].map(inventory_mapping)
old_data['Productsize'] = old_data['Productsize'].map(uofm_mapping)

# 将 'Description' 和 'CropName' 列的内容转换为小写
old_data['Description'] = old_data['Description'].str.lower()
old_data['CropName'] = old_data['CropName'].str.lower()

# 指定列的顺序
columns_order = ['State', 'Customer', 'SRID', 'OrderNbr', 'InventoryID', 'CropName', 'Description', 'Productsize',
                 'Amount', 'Quantity', 'Date', 'PriceClass']
# 输出
old_data = old_data[columns_order]
old_data.to_csv('2018_2024_old.csv', index=False)

```
与新ERP数据拼接
```python
# 加载旧的数据
old_erp = pd.read_csv('2018_2024_old.csv')

# 加载 2024 Sales Data ACU.xlsx
new_erp = pd.read_csv('2024_new.csv')

# 将 'Description' 和 'CropName' 列的内容转换为小写
new_erp['Description'] = new_erp['Description'].str.lower()
new_erp['CropName'] = new_erp['CropName'].str.lower()

# 确保两个数据集的 Date 列为 datetime 类型
old_erp['Date'] = pd.to_datetime(old_erp['Date'], errors='coerce')
new_erp['Date'] = pd.to_datetime(new_erp['Date'], errors='coerce')

# 将 Date 列统一转换为 MM/DD/YYYY 格式
old_erp['Date'] = old_erp['Date'].dt.strftime('%m/%d/%Y')
new_erp['Date'] = new_erp['Date'].dt.strftime('%m/%d/%Y')

# 拼接 old_erp 和 new_erp
combined_erp = pd.concat([old_erp, new_erp], ignore_index=True)
print(combined_erp)

# 输出拼接后的文件
combined_erp.to_csv('combined_sales_data.csv', index=False)

```
数据筛选
重点关注标有重量单位（OZ、LB、种子和公斤）的产品尺寸
```python
# 读取 csv 文件
df = pd.read_csv('combined_sales_data.csv')

# 筛选出 'Productsize' 列中包含 'seed', 'oz', 'lb', 'kg' 的行
keywords = ['seed', 'oz', 'lb', 'kg']
df_filtered = df[df['Productsize'].str.contains('|'.join(keywords), case=False, na=False)]
print(df_filtered)

#输出
df_filtered.to_csv('merged_data.csv', index=False)
```
年份筛选

