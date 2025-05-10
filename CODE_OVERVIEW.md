# Code Overview: Credential Stuffing Detection Implementation

This document provides an overview of the key components and methodology implemented in the credential stuffing detection analysis.

## Data Processing Pipeline

### 1. Data Preparation
```python
# Load and preprocess data
data = pd.read_csv('assignment.csv')
data['datetime'] = pd.to_datetime(data['epoch'], unit='ms')
data.rename(columns={'Unnamed: 0': 'log_id'}, inplace=True)
```

The first step involves loading the dataset and performing basic preprocessing:
- Converting epoch timestamps to datetime objects for easier analysis
- Renaming columns for clarity
- Initial data inspection to understand structure and patterns

### 2. Exploratory Analysis
```python
# Distribution of risk indicators
risk_distribution = data['risk_indication'].value_counts()
risk_distribution_df = risk_distribution.reset_index()
risk_distribution_df.columns = ['Risk Indication', 'Count']
risk_distribution_df['Percentage'] = (risk_distribution_df['Count'] / total_risk_count * 100).round(2)
```

Exploratory analysis focuses on understanding:
- Distribution of existing risk indicators
- Login attempt patterns by origin (country, network type)
- Tool usage distribution
- Status code patterns

### 3. Temporal Analysis
```python
# Analyze login attempts over time
data.set_index('datetime', inplace=True)
data['count'] = 1
login_attempts_by_time = data.resample('T').sum()['count']
```

Temporal analysis is critical for identifying automated attacks:
- Resampling data to different time intervals (seconds, minutes)
- Identifying burst patterns characteristic of scripted attacks
- Analyzing peak login times vs. suspicious activity

### 4. Anomaly Detection
```python
# Calculate login rates by IP
ip_counts = data.groupby(['hashed_ip']).agg({'datetime': ['min', 'max'], 'epoch': 'count'})
ip_counts['duration'] = (ip_counts['datetime']['max'] - ip_counts['datetime']['min']).dt.total_seconds()
ip_counts['rate'] = ip_counts['epoch']['count'] / ip_counts['duration']

# Find suspicious IPs based on threshold
threshold = 0.02  # attempts per second
suspicious_ips = ip_counts[ip_counts['rate'] > threshold].index.tolist()
```

Multiple anomaly detection techniques identify suspicious activity:
- High-frequency login attempts from single IPs
- Unusual status code patterns
- Geographic anomalies
- Tool ID analysis
- Network type analysis

### 5. Risk Classification
```python
# Mark risky login attempts based on detection criteria
data['is_risk'] = (
    (data['hashed_ip'].isin(suspicious_ips))
).astype(bool)
```

The final step creates a classification of potentially malicious login attempts:
- Combining multiple indicators into a single risk flag
- Applying thresholds based on observed patterns
- Creating a labeled dataset for further analysis

### 6. Compromised Credential Detection
```python
# Extract potentially compromised emails
at_risk_data = data[data['is_risk']]
at_risk_emails = at_risk_data['email_hash'].unique()
pd.Series(at_risk_emails).to_csv('at_risk_emails.csv', index=False, header=False)
```

The analysis identifies potentially compromised credentials by:
- Finding successful logins from flagged IP addresses
- Identifying login patterns suggesting credential stuffing
- Creating an exportable list of potentially compromised email hashes

## Visualization Components

The analysis uses multiple visualization techniques to communicate findings:

### Login Attempt Distribution
```python
fig = px.bar(risk_distribution_df,
             x='Risk Indication', 
             y='Count',
             text='Percentage', 
             title='Distribution of Risk Indication',
             color='Risk Indication',  
             color_discrete_map={'True': 'red', 'False': 'green'})
```

### Temporal Patterns
```python
fig = px.line(melted_data, x='datetime', y='count', color='status_code',
              labels={'count': 'Number of Attempts', 'datetime': 'Time', 'status_code': 'Status Code'},
              title='Login Attempts Over Time by Status Code')
```

### Geographic Analysis
```python
fig = px.bar(country_risk_counts,
             y='country', 
             x='count',  
             color='risk_indication',  
             title='Login Attempts by Country with Risk Indication',
             orientation='h')
```

### Tool and Network Analysis
```python
fig = px.bar(filtered_tool_id_counts,
             y='tool_id',  
             x='count',  
             color='risk_indication',  
             title='Top 20 Tool IDs with Risk Indication',
             orientation='h')
```

## Evaluation Methods

The implementation includes basic evaluation of the detection method:

```python
from sklearn.metrics import confusion_matrix, ConfusionMatrixDisplay
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score

# Calculate confusion matrix
cm = confusion_matrix(data['risk_indication'], data['is_risk'])

# Calculate metrics
accuracy = accuracy_score(data['risk_indication'], data['is_risk'])
precision = precision_score(data['risk_indication'], data['is_risk'])
recall = recall_score(data['risk_indication'], data['is_risk'])
f1 = f1_score(data['risk_indication'], data['is_risk'])
```

This provides a basic comparison between the existing risk indication and our custom detection method, though it's important to note that the existing risk indication is just one signal and not a ground truth.