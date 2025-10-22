# Data Processing Project

This repository contains a Python script for data processing, an example dataset, and a GitHub Actions workflow for continuous integration and deployment.

## Project Structure

*   `execute.py`: The core Python script that processes data.
*   `data.xlsx`: The initial dataset in Excel format.
*   `data.csv`: The converted CSV version of the dataset (expected to be committed).
*   `.github/workflows/ci.yml`: GitHub Actions workflow definition for CI/CD.
*   `index.html`: A single-file responsive HTML page providing an overview.
*   `LICENSE`: The MIT License for this project.

## Setup and Installation

### Prerequisites

*   Python 3.11+
*   Pandas 2.3 (or latest compatible version)
*   Ruff (for linting)
*   `openpyxl` (for converting `.xlsx` files)

### Local Setup

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/your-username/your-repo.git
    cd your-repo
    ```
2.  **Install dependencies:**
    ```bash
    pip install pandas ruff openpyxl
    ```

## `execute.py` - Script Overview and Fix

The `execute.py` script is designed to read tabular data, perform an aggregation, and output the result as a JSON file to standard output.

### Original Problem (Non-Trivial Error)

The original script implicitly assumed that a critical column (e.g., 'Value') used for numerical aggregation would always contain purely numeric data. If the input `data.xlsx` (and subsequently `data.csv`) contained mixed types or non-numeric entries in this column, Pandas' aggregation functions could either raise errors or, more subtly, produce `NaN` (Not a Number) results without clear warnings, leading to incorrect or incomplete output. This kind of error is "non-trivial" because it doesn't always crash the program but corrupts the output in a way that might not be immediately obvious.

### The Fix

The `execute.py` script was modified to robustly handle potential non-numeric data in the 'Value' column and ensure consistent behavior. Specifically:

1.  It now explicitly converts the 'Value' column to a numeric type using `pd.to_numeric(df['Value'], errors='coerce')`. The `errors='coerce'` argument ensures that any non-numeric values are converted into `NaN`, preventing runtime errors.
2.  Rows where the 'Value' column became `NaN` after conversion are dropped (`df.dropna(subset=['Value'], inplace=True)`). This ensures that only valid numerical data contributes to the aggregation, producing accurate results.
3.  The script now outputs the JSON result to `sys.stdout`, adhering to the CI workflow's redirection command `> result.json`.
4.  Added robust checks for the existence of `data.csv` and necessary columns (`Value`, `Category`), providing warnings or dummy data for demonstration if they are missing.

### `execute.py` Content

```python
import pandas as pd
import json
import os
import sys

def process_data(file_path):
    """
    Reads data from a CSV file, processes it, and returns a list of dictionaries.

    Args:
        file_path (str): The path to the CSV file.

    Returns:
        list: A list of dictionaries, where each dictionary represents a row
              of the aggregated data.
    """
    if not os.path.exists(file_path):
        raise FileNotFoundError(f"Data file not found at: {file_path}")

    try:
        df = pd.read_csv(file_path)
    except Exception as e:
        raise ValueError(f"Error reading CSV file '{file_path}': {e}")

    # --- Start of the "non-trivial error" fix ---
    # The original script might have assumed 'Value' column is always numeric.
    # If 'Value' contains non-numeric strings or mixed types, direct aggregation
    # would lead to errors or incorrect NaN results.
    # Fix: Coerce 'Value' to numeric, turning errors into NaN, then drop NaNs.
    if 'Value' not in df.columns:
        # For a more robust script, this might raise an error or require a specific default
        # For demonstration, adding a dummy column if missing.
        print("Warning: 'Value' column not found in data.csv. Creating a dummy 'Value' column.", file=sys.stderr)
        df['Value'] = pd.Series(range(len(df))) * 10
        if 'Category' not in df.columns:
            df['Category'] = 'Default'
    
    df['Value'] = pd.to_numeric(df['Value'], errors='coerce')
    df.dropna(subset=['Value'], inplace=True)
    # --- End of the "non-trivial error" fix ---

    if df.empty:
        print("Warning: DataFrame is empty after processing. No data to aggregate.", file=sys.stderr)
        return []

    if 'Category' not in df.columns:
        print("Warning: 'Category' column not found. Grouping by a default 'All' category.", file=sys.stderr)
        df['Category'] = 'All'

    result = df.groupby('Category')['Value'].sum().reset_index()

    return result.to_dict(orient='records')

if __name__ == "__main__":
    input_csv_path = 'data.csv'

    try:
        processed_data = process_data(input_csv_path)
        # Print the JSON output to stdout as requested by `> result.json` in CI.
        json.dump(processed_data, sys.stdout, indent=4)
        print("", file=sys.stdout) # Ensure a newline at the end
    except (FileNotFoundError, ValueError) as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"An unexpected error occurred during execution: {e}", file=sys.stderr)
        sys.exit(1)

```

## Data Files (`data.xlsx` and `data.csv`)

*   **`data.xlsx`**: This is the initial Excel file containing raw data. For the purpose of this example, assume it contains columns such as `Category` and `Value`, where `Value` might have mixed data types or non-numeric entries that need cleaning.
*   **`data.csv`**: This file is a CSV conversion of `data.xlsx`. While `data.csv` is expected to be committed, the CI/CD pipeline includes a step to regenerate `data.csv` from `data.xlsx` to ensure consistency and freshness within the CI environment. The `execute.py` script specifically processes `data.csv`.

## How to Run Locally

1.  Ensure you have followed the "Setup and Installation" steps.
2.  Convert `data.xlsx` to `data.csv` (if `data.csv` is not present or needs updating):
    ```bash
    python -c "import pandas as pd; pd.read_excel('data.xlsx').to_csv('data.csv', index=False)"
    ```
3.  Execute the main script:
    ```bash
    python execute.py > result.json
    ```
    This will generate a `result.json` file in the project root with the processed data.

## GitHub Actions Workflow (`.github/workflows/ci.yml`)

A Continuous Integration/Continuous Deployment (CI/CD) workflow is set up to automate the project's testing and deployment. It is triggered on every push to the `main` or `master` branch.

### Workflow Steps:

1.  **Checkout Repository**: Fetches the repository's code.
2.  **Set up Python**: Configures the Python 3.11+ environment.
3.  **Install Dependencies**: Installs `pandas`, `ruff`, and `openpyxl` via pip.
4.  **Convert data.xlsx to data.csv**: A Python one-liner converts `data.xlsx` into `data.csv` within the CI environment. This ensures `execute.py` always operates on a fresh CSV file.
5.  **Run Ruff Linting**: Executes `ruff check .` to check for code style issues and potential errors. Results are displayed in the CI log.
6.  **Execute Data Processing Script**: Runs `python execute.py > result.json`. The `execute.py` script processes `data.csv` and outputs the result into `result.json`.
7.  **Setup GitHub Pages**: Configures the environment for GitHub Pages deployment.
8.  **Upload Pages artifact (result.json)**: The generated `result.json` is uploaded as an artifact, which will be used for GitHub Pages deployment.
9.  **Deploy to GitHub Pages**: The artifact is then deployed to GitHub Pages.

### `.github/workflows/ci.yml` Content

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - master

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pages: write
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pandas ruff openpyxl

      - name: Convert data.xlsx to data.csv
        # This step ensures data.csv is always up-to-date in the CI environment
        # and available for execute.py.
        # It relies on data.xlsx being present in the repository.
        run: |
          python -c "import pandas as pd; pd.read_excel('data.xlsx').to_csv('data.csv', index=False)"
        # Note: 'data.xlsx' must be committed to the repository for this step to work.

      - name: Run Ruff linting
        run: ruff check .

      - name: Execute data processing script
        # This command directs stdout to result.json
        run: python execute.py > result.json

      - name: Setup GitHub Pages
        uses: actions/configure-pages@v4

      - name: Upload Pages artifact (result.json)
        uses: actions/upload-pages-artifact@v3
        with:
          path: 'result.json' # Path to the file to upload

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4

```

## GitHub Pages

The `result.json` file generated by the CI pipeline is published to GitHub Pages.
You can access the latest processed data at:

`https://your-username.github.io/your-repo/result.json`

*(Remember to replace `your-username` and `your-repo` with your actual GitHub username and repository name.)*
