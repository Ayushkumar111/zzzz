import requests
import os
from datetime import datetime, timedelta
import time
import pandas as pd
import logging

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler("nse_download.log"),
        logging.StreamHandler()
    ]
)

class NSEDataDownloader:
    def __init__(self, download_dir="nse_data"):
        """
        Initialize the NSE data downloader
        
        Args:
            download_dir (str): Directory to save downloaded files
        """
        self.download_dir = download_dir
        self.base_url = "https://www.nseindia.com"
        
        # Create download directory if it doesn't exist
        if not os.path.exists(download_dir):
            os.makedirs(download_dir)
            logging.info(f"Created download directory: {download_dir}")
        
        # Set common headers to mimic browser request
        self.headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36",
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
            "Accept-Language": "en-US,en;q=0.5",
            "Accept-Encoding": "gzip, deflate, br",
            "Connection": "keep-alive",
            "Upgrade-Insecure-Requests": "1",
            "Cache-Control": "max-age=0"
        }
        
        # Create a session to maintain cookies
        self.session = requests.Session()
        self.session.headers.update(self.headers)

    def download_bhavcopy(self, date=None):
        """
        Download the Bhavcopy (daily market data) for a specific date
        
        Args:
            date (datetime, optional): Date for which to download the bhavcopy. Defaults to yesterday.
        
        Returns:
            str: Path to downloaded file or None if download failed
        """
        if date is None:
            date = datetime.now() - timedelta(days=1)
        
        date_str = date.strftime("%d%m%Y")
        file_name = f"cm_bhavcopy_{date_str}.zip"
        download_url = f"https://archives.nseindia.com/content/historical/EQUITIES/{date.strftime('%Y')}/{date.strftime('%b').upper()}/{file_name}"
        
        output_path = os.path.join(self.download_dir, file_name)
        
        try:
            logging.info(f"Downloading Bhavcopy for {date_str} from {download_url}")
            response = self.session.get(download_url, timeout=30)
            
            if response.status_code == 200:
                with open(output_path, 'wb') as f:
                    f.write(response.content)
                logging.info(f"Successfully downloaded to {output_path}")
                return output_path
            else:
                logging.error(f"Failed to download Bhavcopy. Status code: {response.status_code}")
                return None
        except Exception as e:
            logging.error(f"Error downloading Bhavcopy: {str(e)}")
            return None

    def download_option_chain(self, symbol, expiry_date=None):
        """
        Download the option chain data for a specific symbol
        
        Args:
            symbol (str): Stock symbol (e.g., 'NIFTY', 'BANKNIFTY')
            expiry_date (str, optional): Expiry date in format 'DD-MM-YYYY'. Defaults to nearest expiry.
        
        Returns:
            pandas.DataFrame: Option chain data or None if download failed
        """
        try:
            # First, visit the main page to get cookies
            self.session.get("https://www.nseindia.com/option-chain")
            
            # Construct the API URL
            api_url = f"https://www.nseindia.com/api/option-chain-indices?symbol={symbol}"
            if symbol not in ['NIFTY', 'BANKNIFTY', 'FINNIFTY']:
                api_url = f"https://www.nseindia.com/api/option-chain-equities?symbol={symbol}"
            
            logging.info(f"Fetching option chain for {symbol}")
            response = self.session.get(api_url, timeout=30)
            
            if response.status_code == 200:
                data = response.json()
                
                # Save raw JSON
                json_file = os.path.join(self.download_dir, f"{symbol}_option_chain_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json")
                with open(json_file, 'w') as f:
                    import json
                    json.dump(data, f)
                
                # Process and convert to Excel
                records = []
                
                for item in data['records']['data']:
                    if 'CE' in item and 'PE' in item:
                        # Select the right expiry if specified
                        if expiry_date and item['expiryDate'] != expiry_date:
                            continue
                            
                        ce_data = item['CE']
                        pe_data = item['PE']
                        
                        record = {
                            'strikePrice': item['strikePrice'],
                            'expiryDate': item['expiryDate'],
                            'CE_openInterest': ce_data.get('openInterest', 0),
                            'CE_changeinOpenInterest': ce_data.get('changeinOpenInterest', 0),
                            'CE_totalTradedVolume': ce_data.get('totalTradedVolume', 0),
                            'CE_impliedVolatility': ce_data.get('impliedVolatility', 0),
                            'CE_lastPrice': ce_data.get('lastPrice', 0),
                            'CE_change': ce_data.get('change', 0),
                            'PE_openInterest': pe_data.get('openInterest', 0),
                            'PE_changeinOpenInterest': pe_data.get('changeinOpenInterest', 0),
                            'PE_totalTradedVolume': pe_data.get('totalTradedVolume', 0),
                            'PE_impliedVolatility': pe_data.get('impliedVolatility', 0),
                            'PE_lastPrice': pe_data.get('lastPrice', 0),
                            'PE_change': pe_data.get('change', 0)
                        }
                        records.append(record)
                
                # Create DataFrame and save to Excel
                df = pd.DataFrame(records)
                excel_file = os.path.join(self.download_dir, f"{symbol}_option_chain_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx")
                df.to_excel(excel_file, index=False)
                
                logging.info(f"Successfully saved option chain to {excel_file}")
                return df
            else:
                logging.error(f"Failed to download option chain. Status code: {response.status_code}")
                return None
        except Exception as e:
            logging.error(f"Error downloading option chain: {str(e)}")
            return None

    def download_index_data(self, index='NIFTY 50'):
        """
        Download index data
        
        Args:
            index (str): Index name (e.g., 'NIFTY 50', 'NIFTY BANK')
            
        Returns:
            pandas.DataFrame: Index data or None if download failed
        """
        try:
            # Map index names to their NSE identifiers
            index_map = {
                'NIFTY 50': 'NIFTY',
                'NIFTY BANK': 'BANKNIFTY',
                'NIFTY FINANCIAL SERVICES': 'FINNIFTY',
            }
            
            index_code = index_map.get(index, index)
            
            # Visit the main page to get cookies
            self.session.get("https://www.nseindia.com/market-data/live-equity-market")
            
            # Construct the API URL
            api_url = f"https://www.nseindia.com/api/equity-stockIndices?index={index}"
            
            logging.info(f"Fetching data for index {index}")
            response = self.session.get(api_url, timeout=30)
            
            if response.status_code == 200:
                data = response.json()
                
                # Save raw JSON
                json_file = os.path.join(self.download_dir, f"{index_code}_data_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json")
                with open(json_file, 'w') as f:
                    import json
                    json.dump(data, f)
                
                # Create DataFrame from constituent data
                df = pd.DataFrame(data['data'])
                
                # Save to Excel
                excel_file = os.path.join(self.download_dir, f"{index_code}_data_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx")
                df.to_excel(excel_file, index=False)
                
                logging.info(f"Successfully saved index data to {excel_file}")
                return df
            else:
                logging.error(f"Failed to download index data. Status code: {response.status_code}")
                return None
        except Exception as e:
            logging.error(f"Error downloading index data: {str(e)}")
            return None

    def download_corporate_actions(self, symbol):
        """
        Download corporate actions for a specific stock
        
        Args:
            symbol (str): Stock symbol (e.g., 'TCS', 'INFY')
            
        Returns:
            pandas.DataFrame: Corporate actions data or None if download failed
        """
        try:
            # Visit the main page to get cookies
            self.session.get("https://www.nseindia.com/companies-listing/corporate-filings-actions")
            
            # Construct the API URL
            api_url = f"https://www.nseindia.com/api/corporate-actions?index={symbol.upper()}"
            
            logging.info(f"Fetching corporate actions for {symbol}")
            response = self.session.get(api_url, timeout=30)
            
            if response.status_code == 200:
                data = response.json()
                
                # Save raw JSON
                json_file = os.path.join(self.download_dir, f"{symbol}_corporate_actions_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json")
                with open(json_file, 'w') as f:
                    import json
                    json.dump(data, f)
                
                # Create DataFrame
                if data:
                    df = pd.DataFrame(data)
                    
                    # Save to Excel
                    excel_file = os.path.join(self.download_dir, f"{symbol}_corporate_actions_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx")
                    df.to_excel(excel_file, index=False)
                    
                    logging.info(f"Successfully saved corporate actions to {excel_file}")
                    return df
                else:
                    logging.warning(f"No corporate actions found for {symbol}")
                    return None
            else:
                logging.error(f"Failed to download corporate actions. Status code: {response.status_code}")
                return None
        except Exception as e:
            logging.error(f"Error downloading corporate actions: {str(e)}")
            return None

# Example usage
if __name__ == "__main__":
    downloader = NSEDataDownloader()
    
    # Download yesterday's bhavcopy
    bhavcopy_path = downloader.download_bhavcopy()
    
    # Download NIFTY option chain
    nifty_options = downloader.download_option_chain("NIFTY")
    
    # Download index data for NIFTY 50
    nifty_data = downloader.download_index_data("NIFTY 50")
    
    # Download corporate actions for TCS
    tcs_actions = downloader.download_corporate_actions("TCS")
    
    logging.info("All downloads completed.")