const express = require('express');
const bodyParser = require('body-parser');
const fs = require('fs');
const mongoose = require('mongoose');
const ngrok = require('ngrok');
const db = require('./src/db');
const Order = require('./src/order');
const Account = require('./src/data');
const apiKeys = require('./src/apiKeys');

const app = express();
const port = 3000;

function authenticateAPIKey(req, res, next) {
  const apiKey = req.header('X-API-Key') || req.query.api_key;

  if (apiKey && apiKeys[apiKey]) {
    req.apiKey = apiKey; 
    req.apiDataFile = apiKeys[apiKey].file;
    next();
  } else {
    res.status(401).json({ error: 'Unauthorized' });
  }
}

app.use(bodyParser.json());

app.get('/stock/:type', authenticateAPIKey, async (req, res) => {
  const { tk, mk } = req.query;
  const { type } = req.params;

  if (!tk || !mk) {
    return res.status(400).json({ error: 'Thiếu thông tin tk hoặc mk.' });
  }

  try {
    const orderCode = await generateOrderCode();

    const newOrder = new Order({ tk, mk, orderCode, type, apiKey: req.apiKey });
    await newOrder.save();

    res.status(200).json({ orderCode, tk, mk, message: 'Đã lưu thông tin và tạo mã đơn hàng thành công.' });
  } catch (error) {
    console.error('Lỗi khi lưu thông tin và tạo mã đơn hàng:', error);
    res.status(500).json({ error: 'Đã xảy ra lỗi, vui lòng thử lại sau.' });
  }
});

app.get('/account', async (req, res) => {
  const { orderID } = req.query;

  if (!orderID) {
    return res.status(400).json({ error: 'Thiếu thông tin orderID.' });
  }

  try {
    const order = await Order.findOne({ orderCode: orderID }); 

    if (!order) {
      return res.status(404).json({ error: 'Không tìm thấy đơn hàng.' });
    }

    const newAccount = new Account({ tk: order.tk, mk: order.mk, apiKey: order.apiKey }); 
    const apiDataFile = apiKeys[order.apiKey].file; 
    let data;
    try {
      data = JSON.parse(fs.readFileSync(apiDataFile, 'utf8'));
    } catch (error) {
      data = {};
    }

    data.accounts = data.accounts || [];
    data.accounts.push({ tk: order.tk, mk: order.mk });
    fs.writeFileSync(apiDataFile, JSON.stringify(data));

    await newAccount.save();
    await Order.findOneAndDelete({ orderCode: orderID }); 

    res.status(200).json({ tk: order.tk, mk: order.mk, message: 'Nhận tài khoản thành công.' });
  } catch (error) {
    console.error('Lỗi khi lưu thông tin tài khoản:', error);
    res.status(500).json({ error: 'Đã xảy ra lỗi, vui lòng thử lại sau.' });
  }
});

app.get('/clear', authenticateAPIKey, async (req, res) => {
  try {
    const apiDataFile = req.apiDataFile;
    fs.writeFileSync(apiDataFile, JSON.stringify({}));

    res.status(200).json({ message: 'Đã xóa hết dữ liệu trong Order và Account.' });
  } catch (error) {
    console.error('Lỗi khi xóa dữ liệu:', error);
    res.status(500).json({ error: 'Đã xảy ra lỗi, vui lòng thử lại sau.' });
  }
});

app.get('/check', authenticateAPIKey, async (req, res) => {
  try {
    const orders = await Order.find({ type: { $in: ['AD', 'BF'] }, apiKey: req.apiKey }); 

    if (!orders || orders.length === 0) {
      return res.status(404).json({ error: 'Không tìm thấy đơn hàng nào.' });
    }

    let result = {};
    orders.forEach(order => {
      const { type, tk, mk, orderCode } = order;

      if (!result[type]) {
        result[type] = { accounts: [] };
      }

      result[type].accounts.push({ tk, mk, orderCode });
    });

    const apiDataFile = req.apiDataFile;
    let data;
    try {
      data = JSON.parse(fs.readFileSync(apiDataFile, 'utf8'));
    } catch (error) {
      data = {};
    }

    Object.keys(result).forEach(type => {
      data[type] = { accounts: result[type].accounts };
    });

    fs.writeFileSync(apiDataFile, JSON.stringify(data));

    res.status(200).json(result);
  } catch (error) {
    console.error('Lỗi khi lấy thông tin đơn hàng:', error);
    res.status(500).json({ error: 'Đã xảy ra lỗi, vui lòng thử lại sau.' });
  }
});

app.get('/createApiKey/custom', (req, res) => {
  const apiKey = req.query.key;
  
  if (!apiKey) {
    return res.status(400).json({ error: 'Thiếu thông tin key.' });
  }
  
  if (apiKeys[apiKey]) {
    return res.status(400).json({ error: 'API key đã tồn tại.' });
  }
  
  try {
    createApiKey(apiKey, res);
  } catch (error) {
    console.error('Lỗi khi tạo API key:', error);
    res.status(500).json({ error: 'Đã xảy ra lỗi, vui lòng thử lại sau.' });
  }
});

function createApiKey(apiKey, res) {
  const apiDataFile = `./src/api/${apiKey}.json`;
  fs.writeFileSync(apiDataFile, JSON.stringify({}));

  apiKeys[apiKey] = { file: apiDataFile };

  fs.writeFileSync('./src/api/apiKeys.json', JSON.stringify(apiKeys, null, 2)); 

  const apiKeysContent = `const apiKeys = ${JSON.stringify(apiKeys, null, 2)};\n\nmodule.exports = apiKeys;\n`;
  fs.writeFileSync('./src/apiKeys.js', apiKeysContent);

  res.status(200).json({ apiKey });
}

async function generateOrderCode() {
  let orderCode;
  let isUnique = false;

  while (!isUnique) {
    orderCode = generateRandomString(15);
    const existingOrder = await Order.findOne({ orderCode });
    if (!existingOrder) {
      isUnique = true;
    }
  }

  return orderCode;
}

function generateRandomString(length) {
  let result = '';
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  const charactersLength = characters.length;
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * charactersLength));
  }
  return result;
}

app.listen(port, async () => {
  try {
    const url = await ngrok.connect({
      addr: port,
      authtoken: '2j0kaADfSzpsMAcgHbwI0GEeqWO_4KUhcjKGX5yowQJEPgza3',
      region: 'us'
    });

    console.clear();
    console.log(`Server đang chạy tại ${url}`);
    console.log(`API của bạn có thể truy cập tại: ${url}/stock/<type: BF or AD?tk=your tài khoản&mk=your Mật Khẩu>`);
    console.log(`Endpoint kiểm tra: ${url}/check`);
    console.log(`Endpoint tạo API key tùy chỉnh: ${url}/createApiKey/custom?key=<your_custom_key>`);
    console.log(`Endpoint nhận tài khoản: ${url}/account?orderID=<orderID>`);
  } catch (error) {
    console.error('Lỗi khi kết nối ngrok:', error);
  }
});
