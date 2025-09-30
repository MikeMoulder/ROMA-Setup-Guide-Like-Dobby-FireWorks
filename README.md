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
Next is to setup your Docker Compose, in that same docker directory, you'll see a file name called docker-compose.yml, scroll down and add this part to make reference to you caddy file & also your new server which we will complete soon

```
# NEW SERVER PROXY SERVICE
  proxy:
    build:
      context: ../proxy-backend # Adjust path if server.js is elsewhere
      dockerfile: Dockerfile.proxy
    container_name: sentient-proxy
    expose:
      - "4000:4000" # Expose the port defined in server.js (const PORT = 4000)
    # The proxy server needs the .env file to run if it uses environment variables
    env_file:
      - ../.env 
    depends_on:
      - backend # It depends on the backend API it proxies to
      
# CADDY SERVICE
  caddy:
    image: caddy:latest
    container_name: sentient-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - frontend
      - backend
      - proxy # Add dependency on the new proxy service

volumes:
  caddy_data:
  caddy_config:
```
Save it and go back to your "proxy-backend" folder that was created earlier and create a new file and name it "Dockerfile.proxy" and paste this inside it, the name must match "Dockerfile.proxy"

```
# Dockerfile.proxy
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
# Copy the server file
COPY server.js .
# Expose the port (informative only)
EXPOSE 4000
CMD ["node", "server.js"]
```
Save everything and open your terminal and run this to apply changes made
```
cd docker

docker compose -f docker-compose.yml down
docker compose -f docker-compose.yml up -d
```
Use this to view docker logs
```
cd docker
docker compose logs -f
```
Everything should be up and running, you should be able to make calls to your backend using POST method via the URL you created earlier. How? 

## Step 5: Make API Calls To ROMA Like Dobby AI on FireWorks
You can create a new project on your device or anywhere, to make API calls, you can use the method below.
- First to check if server is up and working (server health)

```
const PROXY_IP = 'https://api-roma-sentient-rex.duckdns.org/proxy';
const PROXY_HEALTH_URL = `${PROXY_IP}/health`;

try {
const response = await fetch(PROXY_HEALTH_URL);
if (response.ok) {
    const data = await response.json();
    updateStatus('Connected', 'success', data);
} else {
    const errorData = await response.json();
    updateStatus(`Error ${response.status}`, 'error', errorData);
}
} catch (error) {
updateStatus('Offline', 'error', { error: error.message });
}
```

- Second, to make API request to a prompt, use this
```
const prompt = "your prompt";

const topic = prompt.value.trim();
const PROXY_IP = 'https://api-roma-sentient-rex.duckdns.org/proxy';
const PROXY_RESEARCH_URL = `${PROXY_IP}/research`;

try {
const response = await fetch(PROXY_RESEARCH_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ topic }),
});

if (response.ok) {
    const data = await response.json();
    console.log(data.final_output);
    }
```

That will be all for this guide, if you face any issue, kindly message me on Discord or find me in the builders channel (mikemoulder), also you can test out my ROMA demo setup via this link: https://roma-demo.vercel.app/

Have a nice one everyone cheers.
