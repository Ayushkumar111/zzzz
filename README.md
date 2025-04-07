// app.js
const express = require('express');
const multer = require('multer');
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');
const path = require('path');
const csv = require('csv-parser');

const app = express();
const PORT = process.env.PORT || 3000;

// CSVBox configuration
const CSVBOX_API_KEY = process.env.CSVBOX_API_KEY || 'your_csvbox_api_key';
const CSVBOX_UPLOAD_URL = process.env.CSVBOX_UPLOAD_URL || 'https://app.csvbox.io/api/v1/upload';
const CSVBOX_GET_URL = process.env.CSVBOX_GET_URL || 'https://app.csvbox.io/api/v1/data';
const CSVBOX_TEMPLATE_ID = process.env.CSVBOX_TEMPLATE_ID || 'your_template_id';

// Temporary file path for storing uploaded CSV
const TEMP_CSV_PATH = path.join(__dirname, 'temp_upload.csv');

// MIDDLEWARE FOR FILE SIZE LIMT 
app.use(express.text({ type: 'text/csv', limit: '10mb' }));
app.use(express.json());

// Configure multer for file uploads
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, __dirname);
  },
  filename: function (req, file, cb) {
    cb(null, 'temp_upload.csv');
  }
});
const upload = multer({ storage: storage });

//
 * Upload CSV data to CSVBox
 * @param {string} filePath - Path to the CSV file to upload
 * @returns {Promise<string>} - Promise that resolves with the public URL
 */ SEND KARO DATA TO CSVBOX . FUNCTION 
async function uploadToCSVBox(filePath) {
  try {
    const form = new FormData();
    form.append('file', fs.createReadStream(filePath));
    form.append('api_key', CSVBOX_API_KEY);
    form.append('template_id', CSVBOX_TEMPLATE_ID);

    const response = await axios.post(CSVBOX_UPLOAD_URL, form, {
      headers: {
        ...form.getHeaders(),
      },
      maxBodyLength: Infinity,
      maxContentLength: Infinity,
    });

    if (response.data && response.data.success && response.data.url) {
      return response.data.url;
    } else {
      throw new Error('Failed to get URL from CSVBox response');
    }
  } catch (error) {
    console.error('Error uploading to CSVBox:', error.message);
    throw new Error(`CSVBox upload failed: ${error.message}`);
  }
}

/**
 * Get data from CSVBox
 * @returns {Promise<Array>} - Promise that resolves with data from CSVBox

 */DATA LEKE AO CSV BOX SE . FUNCTION 
async function getDataFromCSVBox() {
  try {
    const response = await axios.get(CSVBOX_GET_URL, {
      params: {
        api_key: CSVBOX_API_KEY,
        template_id: CSVBOX_TEMPLATE_ID
      }
    });

    if (response.data && response.data.success) {
      return response.data.data;
    } else {
      throw new Error('Failed to get data from CSVBox');
    }
  } catch (error) {
    console.error('Error fetching data from CSVBox:', error.message);
    throw new Error(`CSVBox data fetch failed: ${error.message}`);
  }
}

/**
 * Save CSV string data to a file
 * @param {string} csvData - CSV data as string
 * @returns {Promise<string>} - Promise that resolves with the file path

 */
 CSV FILE KA DATA SAVE KRNE KE LIYE 
async function saveCSVToFile(csvData) {
  return new Promise((resolve, reject) => {
    fs.writeFile(TEMP_CSV_PATH, csvData, 'utf8', (err) => {
      if (err) {
        reject(err);
      } else {
        resolve(TEMP_CSV_PATH);
      }
    });
  });
}

/**
 * Clean up temporary files
 * @param {string} filePath - Path to file to delete

 */TEMP FILE JO BNAI THI USKO HTAO 
function cleanupTempFile(filePath) {
  try {
    if (fs.existsSync(filePath)) {
      fs.unlinkSync(filePath);
    }
  } catch (error) {
    console.error('Error cleaning up temp file:', error);
  }
}

MAST API ENDPOINTS BANEGE AB BESS

// POST endpoint to update data with raw CSV content JO MUJHE TU DEGA SCRAPE KRKE 


app.post('/update-data', async (req, res) => {
  try {
    // Check if request body contains CSV data
    if (!req.body || typeof req.body !== 'string' || req.body.trim() === '') {
      return res.status(400).json({
        success: false,
        error: 'Invalid data format. Expected CSV data in request body.'
      });
    }
    
    // Save CSV data to temporary file
    
    const filePath = await saveCSVToFile(req.body);
    
    // Upload to CSVBox
    
    const publicUrl = await uploadToCSVBox(filePath);
    
    // Clean up temporary file
    
    cleanupTempFile(filePath);
    
    // Respond with success and URL
    res.status(200).json({
      success: true,
      message: 'Data uploaded to CSVBox successfully',
      url: publicUrl
    });
  } catch (err) {
    console.error('Error processing CSV data:', err);
    res.status(500).json({
      success: false,
      error: err.message || 'Failed to process and upload CSV data'
    });
  }
});

// Alternative POST endpoint for file upload , YE ALTERNATE METOD HAI JAB TU DIRECT FILE DEDE , EDGE CASE HAI 
app.post('/update-data-file', upload.single('csvFile'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({
        success: false,
        error: 'No file uploaded. Expected a file with field name "csvFile".'
      });
    }
    
    // Upload to CSVBox (file is already saved by multer)
    const publicUrl = await uploadToCSVBox(TEMP_CSV_PATH);
    
    // Clean up temporary file
    cleanupTempFile(TEMP_CSV_PATH);
    
    // Respond with success and URL
    res.status(200).json({
      success: true,
      message: 'Data uploaded to CSVBox successfully',
      url: publicUrl
    });
  } catch (err) {
    console.error('Error processing CSV file:', err);
    res.status(500).json({
      success: false,
      error: err.message || 'Failed to process and upload CSV file'
    });
  }
});

GET KA ALTHOUGH UTNA KAAM NHI HAI PAR DATA RETREIVE KRNE KA MAAN KAR GYA CSV BOX SE TAB USKE LIYE HAI 


// GET endpoint to retrieve data from CSVBox
app.get('/get-data', async (req, res) => {
  try {
    const data = await getDataFromCSVBox();
    res.status(200).json({
      success: true,
      data: data
    });
  } catch (err) {
    console.error('Error fetching data:', err);
    res.status(500).json({
      success: false,
      error: err.message || 'Failed to fetch data from CSVBox'
    });
  }
});



// Health check endpoint
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

// Start server
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

module.exports = app; // For testing purposes
