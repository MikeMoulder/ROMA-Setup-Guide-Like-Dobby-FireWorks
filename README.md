# Ultimate Guide To Setup ROMA With API Features Like Dobby FireWorks

## Step 1 : ROMA Setup
First, you need to setup your ROMA, I'll suggest you use a VPS as it is easier than a local setup, you can use the guide below to achieve that.
- ROMA VPS: https://github.com/MikeMoulder/ROMA-VPS-Guide-With-UI-UX/edit/main/README.md
Also, for cheaper models, it is advisable you migrate from Openrouter to Fireworks, follow this guide from this amazing expert to achieve that.
- https://github.com/programmerer1/sentient-roma-fireworks-api

## Step 2: Get A DNS For Your VPS
First, copy the IP Address of your VPS (the same one you use SSH with), then visit https://www.duckdns.org/ to create a free domain for your VPS IP. It is easy to navigate, just sign up and enter your desired domain name and paste your VPS IP Address in the current ip field and save. ![duckdns](duckdns.png)

## Step 3: Setup An Intermediary Server
Usually, you will face a lot of issues with making direct calls to your ROMA backend at http:localhost:5000, especially with CORS, also you won't have a room to be creative with things, that's the reason for this intermediary server thing.

To do that, open your terminal and paste this command
```
mkdir proxy-backend
cd proxy-backend
npm init -y
npm install express node-fetch cors
```
You should see a new directory called "proxy-backend", then inside it create a new script and call it "server.js" and open the script and paste this
```
const express = require('express');
const cors = require('cors');

const app = express();
const PORT = 4000;
// Keep the target API URL as defined
const TARGET_API_URL = 'http://backend:5000/api/simple/execute';

// --- CORS FIX ---
// The frontend is running on a DIFFERENT device (e.g., a laptop accessing the server by IP).
// We must reliably allow ALL origins (*) during testing to prevent CORS issues
// caused by the IP/hostname mismatch between the client (frontend) and the server (proxy).

// Temporarily allow all origins with the simplest configuration.
// In production, you would replace '*' with the specific domain of your deployed frontend.
app.use(cors({
    origin: '*',
    methods: ['GET', 'POST'],
}));

app.use(express.json());
// ----------------

// HEALTH CHECK ROUTE
app.get('/health', (req, res) => {
    console.log('Health check received.');
    res.json({ status: 'Proxy Server is Running', code: 200, message: 'Ready to relay requests.' });
});

// MAIN PROXY ENDPOINT
app.post('/proxy/research', async (req, res) => {
    const incomingBody = req.body;

    // --- PAYLOAD TRANSFORMATION FIX ---
    // The ROMA API requires "goal" and likely a "config" object, even for simple execute.
    const payloadToSend = {
        goal: incomingBody.topic || incomingBody.goal || 'No topic provided',
        max_steps: 1
    };

    console.log(`[Proxy] Transforming payload and forwarding to ${TARGET_API_URL}`);
    // ----------------------------------

    try {
        // Note: The 'fetch' here is server-to-server and is NOT subject to CORS rules.
        const apiResponse = await fetch(TARGET_API_URL, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payloadToSend), // Use the structured payload
        });

        if (!apiResponse.ok) {
            const errorText = await apiResponse.text();
            console.error(`Target API responded with status ${apiResponse.status}: ${errorText}`);
            return res.status(apiResponse.status).json({
                error: 'Target API request failed',
                details: errorText.length > 500 ? 'Response too large to display.' : errorText
            });
        }

        const data = await apiResponse.json();
        res.json(data);

    } catch (error) {
        console.error('Proxy Error (Network/Internal):', error);
        res.status(500).json({ error: 'Internal Server Error during proxy request', details: error.message });
    }
});

// Start the server
app.listen(PORT, () => {
    console.log(`Proxy server running on http://localhost:${PORT}`);
});

```
## Step 4: Setup Caddy & Docker Compose
Navigate to your docker folder and create a new filename "Caddyfile" no extension just pure "Caddyfile" and paste this, remeber to replace <DNS_URL> with the one you created on DuckDNS in Step 2

```
<DNS_URL> {
    handle_path /* {
        reverse_proxy sentient-proxy:4000
    }
}
```
