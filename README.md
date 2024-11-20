fetching data script
import csv
import os
import gzip
import shutil
import logging
from iqoptionapi.stable_api import IQ_Option
import time
from datetime import datetime

# Configure logging
logging.basicConfig(filename='data_collection.log', level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Constants
EMAIL = "email"  # Replace with your email
PASSWORD = "password"     # Replace with your password
CURRENCY_PAIR = "EURUSD"       # The asset for which to fetch data
CANDLE_DURATION = 60           # Duration of each candlestick in seconds
NUMBER_OF_CANDLES = 500        # Number of candles to fetch
MAX_REQUESTS = 70              # Total number of requests
REQUEST_DELAY = 1              # Delay between requests in seconds
BATCH_SIZE = 10                # Number of requests per batch
BATCH_DELAY = 5                # Delay after a batch of requests
CSV_FILE_NAME = 'candlestick_data.csv'  # CSV file name

# Initialize API
Iq = IQ_Option(EMAIL, PASSWORD)
if not Iq.check_connect():
    logging.error("Failed to connect to IQ Option API.")
    raise Exception("Connection to IQ Option API failed.")

# Function to write data to CSV
def write_to_csv(file_name, data, mode='a'):
    if mode == 'w' or os.stat(file_name).st_size == 0:
        with open(file_name, mode, newline='') as file:
            writer = csv.writer(file)
            writer.writerow(['Date', 'Open', 'Close', 'High', 'Low'])  # Candlestick data header
    with open(file_name, mode, newline='') as file:
        writer = csv.writer(file)
        writer.writerows(data)

# Function to compress the CSV file
def compress_csv(file_name):
    with open(file_name, 'rb') as f_in:
        with gzip.open(file_name + '.gz', 'wb') as f_out:
            shutil.copyfileobj(f_in, f_out)
    os.remove(file_name)  # Remove the original file after compression

# Function to validate data structure
def validate_candle_data(data):
    required_keys = {'from', 'open', 'close', 'high', 'low'}
    return isinstance(data, list) and all(required_keys.issubset(candle.keys()) for candle in data)

# Function to fetch candlestick data
def fetch_candlestick_data(asset, end_from_time):
    all_data = []
    for i in range(0, MAX_REQUESTS, BATCH_SIZE):
        data = Iq.get_candles(asset, CANDLE_DURATION, min(BATCH_SIZE, NUMBER_OF_CANDLES - i), end_from_time)
        if not validate_candle_data(data):
            logging.error("Invalid data format received from API.")
            return []
        
        formatted_data = [
            [datetime.fromtimestamp(candle['from']).strftime('%Y-%m-%d %H:%M:%S'), candle['open'], candle['close'], candle['high'], candle['low']]
            for candle in data
        ]
        all_data.extend(formatted_data)
        logging.info(f"Fetched {len(data)} candles in batch starting from index {i}.")
        
        time.sleep(REQUEST_DELAY)  # Delay between requests
        if (i + BATCH_SIZE) < MAX_REQUESTS:
            time.sleep(BATCH_DELAY)  # Delay after a batch of requests

    return all_data

# Main data fetching logic
def main():
    asset = CURRENCY_PAIR
    end_from_time = time.time()

    try:
        formatted_data = fetch_candlestick_data(asset, end_from_time)
        if formatted_data:
            write_to_csv(CSV_FILE_NAME, formatted_data)
            logging.info(f"Successfully wrote {len(formatted_data)} candles to {CSV_FILE_NAME}.")
        else:
            logging.warning("No valid data to write to CSV.")

    except Exception as e:
        logging.error(f"An error occurred during data fetching: {e}")

    finally:
        compress_csv(CSV_FILE_NAME)
        logging.info(f"Compressed {CSV_FILE_NAME} to {CSV_FILE_NAME}.gz")
        print(f"Compressed {CSV_FILE_NAME} to {CSV_FILE_NAME}.gz")

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        logging.info("Data collection interrupted by user.")
        print("Data collection interrupted by user.")

