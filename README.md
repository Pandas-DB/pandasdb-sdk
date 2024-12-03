# pandas-db-sdk

[![PyPI version](https://badge.fury.io/py/pandas-db-sdk.svg)](https://badge.fury.io/py/pandas-db-sdk)
[![Python Versions](https://img.shields.io/pypi/pyversions/pandas-db-sdk.svg)](https://pypi.org/project/pandas-db-sdk/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A Python SDK for efficiently storing and managing pandas DataFrames in AWS S3, with automatic versioning, chunking, and optimized storage patterns. Think of it as a "pandas-native database" that automatically handles partitioning, versioning, and efficient storage of your DataFrames.

## Features

- **Pandas-Native**: Works seamlessly with pandas DataFrames
- **Automatic Partitioning**: Smart chunking based on DataFrame size
- **Version Control**: Built-in versioning with optional retention policies
- **Storage Patterns**:
  - Date-based partitioning (`date_column: 'Date'`)
  - ID-based range partitioning (`id_column: 'ID'`)
  - Custom external keys
  - Timestamp-based versioning
- **Performance**: 
  - Automatic compression (.csv.gz)
  - Chunking of large DataFrames (100K rows per chunk)
  - Parallel processing capabilities
- **Security**: 
  - User isolation (separate S3 buckets)
  - AWS Cognito authentication
  - Fine-grained access control
- **Flexible Authentication**:
  - Username/password authentication
  - Token-based authentication
  - Automatic token refresh

## Installation

```bash
pip install pandas-db-sdk
```

## Quick Start

```python
from pandas_db_sdk.client import DataFrameClient
import pandas as pd

# Initialize client with username/password
client = DataFrameClient(
    api_url='your-api-url',
    user='your-username',
    password='your-password'
)

# Or initialize with an existing token
client = DataFrameClient(
    api_url='your-api-url',
    auth_token='your-cognito-token'
)

# Create sample DataFrame
df = pd.DataFrame({
    'date': ['2024-01-01', '2024-01-02'],
    'id': [1, 2],
    'value': [100, 200]
})

# Store DataFrame (automatically partitioned by date)
client.load_dataframe(
    df=df,
    dataframe_name='my-project/dataset1',
    columns_keys={'date': 'Date'}
)

# Retrieve DataFrame
df_retrieved = client.get_dataframe('my-project/dataset1')
```

## Authentication

Three ways to authenticate:

1. **Username/Password**:
```python
client = DataFrameClient(
    api_url='your-api-url',
    user='your-username',
    password='your-password'
)
```

2. **Existing Token**:
```python
client = DataFrameClient(
    api_url='your-api-url',
    auth_token='your-cognito-token'
)
```

3. **Get Token Only**:
```python
token = DataFrameClient.get_auth_token(
    api_url='your-api-url',
    user='your-username',
    password='your-password'
)
```

## Key Concepts

### DataFrame Names and Paths
DataFrames can be organized hierarchically:
```python
'project/dataset'           # Two levels
'project/dataset/subset'    # Three levels
'single_name'              # Single level
```

### Storage Keys
Two types of keys determine how your data is stored:

1. **Column Keys**:
  - `Date`: Partitions data by date
  - `ID`: Partitions data by ID ranges

2. **External Keys**:
  - `NOW`: Automatic timestamp-based versioning
  - Custom keys: Your own versioning scheme

### Storage Methods
1. **Concat** (default):
  - Preserves existing data and appends new data 
  - Maintains unaffected partitions unchanged

2. **keep_last**:
  - Replaces data only where new data exists
  - Preserves data in unaffected partitions

## Usage Examples

### 1. Time Series Data
```python
# Concat mode (default)
client.load_dataframe(
   df,
   'timeseries/daily_metrics',
   columns_keys={'timestamp': 'Date'}
)

# Replace mode (only those dates that collide)
client.load_dataframe(
   df,
   'timeseries/daily_metrics',
   columns_keys={'timestamp': 'Date'},
   storage_method='keep_last'
)
```

### 2. User Data with ID Ranges

```python
# Concat mode with ID partitioning
client.load_dataframe(
    df,
    'users/profiles',
    columns_keys={'user_id': 'ID'}
)

# Replace mode (those ID ranges or Dates that collide) with multiple partitions
client.load_dataframe(
    df,
    'transactions',
    columns_keys={
        'transaction_date': 'Date',
        'customer_id': 'ID'
    },
    storage_method='keep_last'
)
```

### 3. Version Management

```python
# Keep only latest version
client.load_dataframe(
    df,
    'reports/daily',
    external_key='NOW',
    keep_last=True
)
```

### 4. Multiple Partitioning Keys

```python
# Partition by both date and ID
client.load_dataframe(
    df,
    'transactions',
    columns_keys={
        'transaction_date': 'Date',
        'customer_id': 'ID'
    }
)
```

## API Reference

### DataFrameClient

```python
class DataFrameClient:
    """Client for managing pandas DataFrames in cloud storage"""
    
    @staticmethod
    def get_auth_token(
        api_url: str,
        user: str,
        password: str,
        region: str = 'eu-west-1'
    ) -> str:
        """
        Get authentication token using username and password
        
        Returns:
            Authentication token string
        """
    
    def load_dataframe(
        self,
        df: Union[pd.DataFrame, str, Dict],
        dataframe_name: str,
        columns_keys: Optional[Dict[str, str]] = None,
        external_key: str = 'NOW',
        keep_last: bool = False
    ) -> Dict:
        """
        Store a DataFrame with optional partitioning and versioning.
        
        Args:
            df: DataFrame to store (DataFrame, JSON string, or dict)
            dataframe_name: Name/path for storage
            columns_keys: Column partitioning settings
            external_key: Version identifier
            keep_last: Whether to keep only latest version
            
        Returns:
            Metadata dictionary with storage information
        """
        
    def get_dataframe(
        self,
        dataframe_name: str,
        external_key: Optional[str] = None,
        use_last: bool = False
    ) -> pd.DataFrame:
        """
        Retrieve a stored DataFrame.
        
        Args:
            dataframe_name: Name/path of DataFrame
            external_key: Optional version filter
            use_last: Whether to get only latest version
            
        Returns:
            Retrieved DataFrame
        """
        
    def delete_dataframe(
        self,
        dataframe_name: str,
        external_key: Optional[str] = None
    ) -> Dict[str, Any]:
        """
        Delete a stored DataFrame.
        
        Args:
            dataframe_name: Name/path of DataFrame
            external_key: Optional version filter
            
        Returns:
            Deletion confirmation
        """
```

## Storage Structure

### Date Partitioning
```
bucket/
└── dataframe_name/
    └── date_column/
        └── 2024-01-01/
            └── chunk_uuid.csv.gz
```

### ID Partitioning
```
bucket/
└── dataframe_name/
    └── id_column/
        └── from_1000_to_2000/
            └── chunk_uuid.csv.gz
```

### Version Control
```
bucket/
└── dataframe_name/
    └── external_key/
        └── default/
            ├── 2024-01-01/
            │   └── 123456_chunk_uuid.csv.gz
            └── last_key.txt
```

## Error Handling

The SDK provides custom exceptions for better error handling:

```python
try:
    client.load_dataframe(df, 'test/df1')
except AuthenticationError as e:
    print(f"Authentication failed: {e}")
except APIError as e:
    print(f"API call failed: {e}")
```

## Requirements

- Python ≥ 3.9
- pandas ≥ 1.0.0
- requests ≥ 2.25.0
- boto3 ≥ 1.26.0

## AWS Setup

This SDK requires AWS infrastructure:
1. API Gateway endpoint
2. Cognito User Pool
3. Lambda functions
4. S3 storage
5. DynamoDB metadata store

Contact your AWS administrator for credentials and endpoints.

## Development

```bash
# Clone repository
git clone https://github.com/yourusername/pandas-db-sdk.git
cd pandas-db-sdk

# Install development dependencies
pip install -e ".[dev]"

# Run tests
pytest

# Build distribution
python setup.py sdist bdist_wheel
```

## Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open Pull Request

## License

[MIT License](LICENSE)

## Support

- GitHub Issues: [pandas-db-sdk/issues](https://github.com/yourusername/pandas-db-sdk/issues)
- Documentation: [pandas-db-sdk.readthedocs.io](https://pandas-db-sdk.readthedocs.io)
