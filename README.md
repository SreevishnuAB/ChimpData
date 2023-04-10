# Chimp Data

## Data pre-processing (Proximity_Data_(Jan_Aug).ipynb)

---

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

## Computing PAI

---

- Fill empty cells with empty string (for easier comparisons necessary down the line)

```Python
df_pai = df_pai.fillna('')
```

- Rename columns with terms from PAI formula for ease of understanding

```Python
df_pai = df_pai.rename(columns={"fuata_follow": "A", "neighbours_normal:nearest_neighbour": "B", "neighbours_normal:other_neighbours": "party"})
```

- Uniformize values

```Python
df_pai['A'] = df_pai['A'].str.lower()
df_pai['B'] = df_pai['B'].str.lower()
df_pai['party'] = df_pai['party'].str.lower()
```

- Clean names of inviduals in columns A and B

```Python
valid_names: List = list(set(df_pai['A'].unique()))
valid_names.remove("unknown")
.
.
.
to_be_cleaned = list(set(df_pai['B'].unique()) - set(df_pai['A'].unique()))
to_be_cleaned.sort(key=len)
to_be_cleaned
.
.
.
valid_names.extend(to_be_cleaned[0:6])
.
.
.
df_pai.loc[df_pai['B'] == "bingwabingwa", 'B'] = "bingwa"
.
.
.
df_clean = pd.DataFrame()

for name in valid_names:
  df_clean = pd.concat([df_clean, df_pai[df_pai['B'] == name]])
.
.
.
df_pai_wo_0: pd.DataFrame = df_clean.loc[(df_clean['A'] != '0') & (df_clean['B'] != '0'), :]
df_pai_wo_0 = df_pai_wo_0[~((df_pai_wo_0['A'] == 'sanaa') & (df_pai_wo_0['B'] == 'sanaa'))]
df_pai_wo_0 = df_pai_wo_0[~((df_pai_wo_0['A'] == 'kinanda') & (df_pai_wo_0['B'] == 'kinanda'))]
df_pai_wo_0 = df_pai_wo_0[~((df_pai_wo_0['A'] == 'kinanda') & (df_pai_wo_0['B'] == 'kinanda'))]
df_pai_wo_0 = df_pai_wo_0[~((df_pai_wo_0['A'] == 'unknown') | (df_pai_wo_0['B'] == 'unknown'))]
```

- Add focal and nearest neighbours to party, for ease of comparison

```Python
df_pai_wo_0['party'] = df_pai_wo_0['A'] + ',' + df_pai_wo_0['B'] + ',' + df_pai_wo_0['party']
```

- Separate wet and dry season data

```Python
df_pai_wo_0['Month'] = pd.to_numeric(df_pai_wo_0['Month'])
df_pai_wet_season = df_pai_wo_0[(df_pai_wo_0['Month'] >= 11) | (df_pai_wo_0['Month'] <= 4)]
df_pai_dry_season = df_pai_wo_0[(df_pai_wo_0['Month'] >= 5) & (df_pai_wo_0['Month'] <= 10)]
```

- Find unique pairs for each season

```Python
gpby_wet_season = df_pai_wet_season.groupby(['A', 'B'])
unique_pairs_wet_season = map(lambda item: frozenset(item), list(gpby_wet_season.groups.keys()))
list_unique_pairs_wet_season: List = [list(i) for i in set(unique_pairs_wet_season)]
list_unique_pairs_wet_season.sort(key=lambda x: x[0])
unique_pair_count_wet_season: Dict = {}

for a, b in list_unique_pairs_wet_season:
  if a not in unique_pair_count_wet_season:
    unique_pair_count_wet_season[a] = {}
  unique_pair_count_wet_season[a][b] = 0
```

- Find unique individuals for each season

```Python
unique_individuals_wet_season = set()

for pairs in list_unique_pairs_wet_season:
  unique_individuals_wet_season.update(pairs)

list_unique_individuals_wet_season = list(unique_individuals_wet_season)
```

- Compute frequency of occurence of each unique individual for each party size they were seen in

```Python
for individual in unique_individuals_wet_season:
   for party in list_party_wet_season:
    #  print(party)
     if individual in party:
       party_size = len(party)
       if party_size in dict_party_freq_wet_season[individual].keys():
         dict_party_freq_wet_season[individual][party_size] += 1
       else:
         dict_party_freq_wet_season[individual][party_size] = 1
```

- Create look-up table for denominator terms

```Python
n_total_neighbours_individual_wet_season = {}

for individual, party_info in dict_party_freq_wet_season.items():
  t = 0
  for party_size, n_occurences in party_info.items():
    t += (n_occurences * (party_size - 1))
  n_total_neighbours_individual_wet_season[individual] = t
```

- Calculate numerator term 2

```Python
n_total_neighbours_all_wet_season = 0
for party in list_party_wet_season:
  party_size = len(party)
  n_total_neighbours_all_wet_season += (party_size * (party_size - 1))
```

- Compute the occurence of each unique pair for each season

```Python
for a, b in list_unique_pairs_wet_season:
  count = 0
  for party in list_party_wet_season:
    if a in party and b in party:
      count += 1
  if a not in pair_occurrence_wet_season.keys():
    pair_occurrence_wet_season[a] = {}
  pair_occurrence_wet_season[a][b] = count
```

- Calculate PAI for each pair A - B

```Python
pai_by_pairs_wet_season: List = []

for a, b in list_unique_pairs_wet_season:
  iab = pair_occurrence_wet_season[a][b]
  n_ai = n_total_neighbours_individual_wet_season[a]
  n_bi = n_total_neighbours_individual_wet_season[b]
  pai = (iab * n_total_neighbours_all_wet_season)/(n_ai * n_bi)
  pai_by_pairs_wet_season.append({'A': a, 'B': b, "PAI": pai})
```

## Creating Network Graphs using PAI

---

- Create pair - PAI tuples for creating edges

```Python
pai_values_wet_season = list(map(lambda x: list(x), df_wet_season.values))
pai_values_dry_season = list(map(lambda x: list(x), df_dry_season.values))

edge_data_wet_season = [(a, b, {"weight": pai}) for a, b, pai in pai_values_wet_season]
edge_data_dry_season = [(a, b, {"weight": pai}) for a, b, pai in pai_values_dry_season]
```

- Adjust edge opacity according to weight

```Python
edge_opacity_wet_season = [weight/weights_wet_season_copy[-1] for weight in weights_wet_season]
```

- Scale weights for the purpose of representation

```Python
weight_adjusted_wet_season = list(map(lambda x: x * 2, weights_wet_season))
```

- Clustering coefficient and mean clustering coefficient are available out of the box with NetworkX
