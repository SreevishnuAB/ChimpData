# Data pre-processing (Proximity_Data_(Jan_Aug).ipynb)

- Add year column to the read data

```Python
df1['Year'] = "2021"
df2['Year'] = "2020"
```

- Exclude irrelevant columns

```Python
df1 = df1[["Year", "Month", "Day", "scan_time", "fuata_follow", "neighbours_normal:nearest_neighbour", "neighbours_normal:other_neighbours"]]
df2 = df2[["Year", "Month", "Day", "scan:time", "individual:id_focal", "nearest_neighbour", "neighbours:other_neighbours"]]
```

- Standardise column names

```Python
df1 = df1.rename(columns={"Date": "Day"})
.
.
.
columns_df1 = list(df1.columns)
columns_df2 = list(df2.columns)

rename_dict = {k: v for k, v in zip(columns_df2, columns_df1)}

df2 = df2.rename(columns=rename_dict)
```

- Merge data

```Python
df = pd.concat([df1, df2], ignore_index=True)
```

- Uniformize date and month values

```Python
df['Month'] = df['Month'].str.zfill(2)
df['Day'] = df['Day'].str.zfill(2)
```

- Remove empty values

```Python
df_cleaned = df[~df['fuata_follow'].isnull()]
df_cleaned = df_cleaned[~(df['neighbours_normal:nearest_neighbour'].isnull() & df['neighbours_normal:other_neighbours'].isnull())]
```

- Group data by required fields

```Python
gpby_with_nn = df_cleaned.groupby(['fuata_follow', 'neighbours_normal:nearest_neighbour', 'Date', 'hour'])

df_grouped_with_nn = pd.DataFrame()
for _, group in gpby_with_nn:
  df_grouped_with_nn = pd.concat([df_grouped_with_nn, group])
```

- For all groups created above, merge all nearest neighbours and other neighbours in each group

```Python
  set_on = set(group.loc[~group['neighbours_normal:other_neighbours'].isna(), 'neighbours_normal:other_neighbours'].values)
  set_nn = set(group.loc[~group['neighbours_normal:nearest_neighbour'].isna(), 'neighbours_normal:nearest_neighbour'].values)
  set_on = set_on - set_nn
  first_row = group.iloc[0]
  first_row['neighbours_normal:other_neighbours'] = ','.join(str(v) for v in list(set_on))
  # print(first_row)
  list_processed_nn.append(first_row)
```
