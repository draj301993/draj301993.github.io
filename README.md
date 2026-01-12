<apex:page controller="ChatFlowController">
  <style>
    /* Your existing styles */
    #chat-widget {
      position: fixed;
      bottom: 20px;
      right: 20px;
      width: 360px;
      background: #fff;
      border-radius: 16px;
      box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);
      font-family: Arial, sans-serif;
      z-index: 9999;
      overflow: hidden;
      transition: all 0.3s ease-in-out;
    }

    .chat-header {
      background: rgb(137, 211, 41);
      color: white;
      padding: 14px 18px;
      font-size: 16px;
      font-weight: bold;
      text-align: center;
    }

    #launchChatBtn {
      display: none;
      position: fixed;
      bottom: 100px;
      right: 20px;
      padding: 12px 24px;
      background: rgb(137, 211, 41);
      color: white;
      border: none;
      border-radius: 8px;
      font-size: 14px;
      font-weight: bold;
      cursor: pointer;
      box-shadow: 0 4px 10px rgba(0, 0, 0, 0.15);
      z-index: 9998;
    }

    #launchChatBtn:hover {
      background: rgb(120, 190, 35);
    }

    #availabilityMessage {
      padding: 18px;
      text-align: center;
      color: #333;
      font-size: 14px;
    }

    .error-message {
      color: #d32f2f;
    }

    .success-message {
      color: #388e3c;
    }
  </style>

  <div id="chat-widget">
    <div id="chat-header" class="chat-header">💬 Checking availability...</div>
    <p id="availabilityMessage">Please wait...</p>
  </div>

  <button id="launchChatBtn" onclick="document.dispatchEvent(new Event('launchMIAWChatEvent'))">
    Chat with a Specialist
  </button>

  <div id="embedded-messaging-root"></div>
  <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
  <script type="text/javascript">
    (function() {
      'use strict';

      // IMPORTANT: These credentials should NOT be in client-side code
      // They should be handled by your Apex controller instead
      // This is a security risk - move OAuth logic to server-side
      
      const SALESFORCE_CONFIG = {
        instanceUrl: 'https://bayeragmiidas--test.sandbox.my.salesforce.com',
        apiEndpoint: '/services/apexrest/CycleExample'
      };

      // Global state
      let authState = {
        accessToken: null,
        refreshToken: null,
        tokenExpiry: null
      };

      // Initialize stored tokens (in production, use secure storage)
      function initializeAuth() {
        // In a real implementation, these would come from secure server-side session
        authState.accessToken = '00D9O000005Id3K!AQEAQBt3Vk7uzx3b3q7DZPtwLMhhT3cPGwDQ..6zBkan9zqHzLYOotWoFHYq9APpco5Vmy43CDo3xvVYlRbN4pYrUgZ.xD6o';
        authState.refreshToken = '5Aep861pw2VNBY3IWYyH3ybCjG64bbZTO4o9DUVi3s6VCellJnFOOOsQ8nhAh5V1NXBEAmkfpu_ncGXHPjU_JYd';
      }

      // Function to check if token is expired
      function isTokenExpired(accessToken) {
        if (!accessToken) return true;
        
        // Salesforce session tokens don't always follow JWT format
        // Check if token has JWT format (3 parts separated by dots)
        const parts = accessToken.split('.');
        if (parts.length === 3) {
          try {
            const payload = JSON.parse(atob(parts[1]));
            const expirationTime = payload.exp * 1000;
            return Date.now() > expirationTime;
          } catch (e) {
            console.warn('Could not parse token as JWT:', e);
            // If we can't parse it, assume it might be expired and try to use it anyway
            return false;
          }
        }
        
        // For non-JWT tokens, we'll try to use them and handle 401 errors
        return false;
      }

      // Refresh token function
      async function refreshAccessToken() {
        // SECURITY WARNING: This should be done server-side via Apex controller
        // For now, commenting out as it exposes client secret
        console.warn('Token refresh should be handled server-side');
        throw new Error('Token refresh must be implemented server-side in Apex controller');
      }

      // Main function to query Salesforce
      async function querySalesforceAccount() {
        const availabilityMsg = document.getElementById('availabilityMessage');
        const chatHeader = document.getElementById('chat-header');
        const launchBtn = document.getElementById('launchChatBtn');

        try {
          // Check if we have a token
          if (!authState.accessToken) {
            availabilityMsg.innerHTML = '<span class="error-message">Authentication required. Please contact administrator.</span>';
            chatHeader.textContent = '⚠️ Authentication Error';
            return;
          }

          // Make the API call
          const endpoint = SALESFORCE_CONFIG.instanceUrl + SALESFORCE_CONFIG.apiEndpoint;
          console.log('Calling endpoint:', endpoint);

          const response = await fetch(endpoint, {
            method: 'GET',
            headers: {
              'Authorization': `Bearer ${authState.accessToken}`,
              'Content-Type': 'application/json',
              'Accept': 'application/json'
            }
          });

          console.log('Response status:', response.status);
          console.log('Response headers:', response.headers);

          // Handle different response statuses
          if (response.status === 401) {
            availabilityMsg.innerHTML = '<span class="error-message">Session expired. Please refresh the page.</span>';
            chatHeader.textContent = '⚠️ Session Expired';
            console.error('Unauthorized - token may be expired');
            return;
          }

          if (response.status === 403) {
            availabilityMsg.innerHTML = '<span class="error-message">Access denied. Insufficient permissions.</span>';
            chatHeader.textContent = '⚠️ Access Denied';
            return;
          }

          if (response.status === 404) {
            availabilityMsg.innerHTML = '<span class="error-message">Service endpoint not found.</span>';
            chatHeader.textContent = '⚠️ Service Error';
            return;
          }

          if (response.status === 500) {
            availabilityMsg.innerHTML = '<span class="error-message">Server error. Please try again later.</span>';
            chatHeader.textContent = '⚠️ Server Error';
            return;
          }

          // Get response text first
          const responseText = await response.text();
          console.log('Raw response:', responseText);

          // Check if response is empty
          if (!responseText || responseText.trim() === '') {
            console.error('Empty response from server');
            availabilityMsg.innerHTML = '<span class="error-message">No data received from server.</span>';
            chatHeader.textContent = '⚠️ No Data';
            return;
          }

          // Try to parse JSON
          let data;
          try {
            data = JSON.parse(responseText);
            console.log('Parsed data:', data);
          } catch (parseError) {
            console.error('JSON parse error:', parseError);
            console.error('Response was:', responseText);
            availabilityMsg.innerHTML = '<span class="error-message">Invalid response format from server.</span>';
            chatHeader.textContent = '⚠️ Data Error';
            return;
          }

          // Process the successful response
          if (data) {
            window.isWithinBH = data.isWithinBH;
            window.availability = data.Availability;
            window.numberOfAgents = data.numberofAgent || 0;

            console.log('Agents available:', window.numberOfAgents);
            console.log('Within business hours:', window.isWithinBH);

            // Update UI based on availability
            if (window.numberOfAgents > 0 && window.isWithinBH) {
              launchBtn.style.display = 'block';
              availabilityMsg.innerHTML = `<span class="success-message">✓ ${window.numberOfAgents} agent(s) available</span>`;
              chatHeader.textContent = '💬 Chat Available';
            } else if (!window.isWithinBH) {
              launchBtn.style.display = 'none';
              availabilityMsg.innerHTML = '<span>We\'re currently outside business hours. Please check back later.</span>';
              chatHeader.textContent = '🕐 Outside Business Hours';
            } else {
              launchBtn.style.display = 'none';
              availabilityMsg.innerHTML = '<span>All agents are currently busy. Please try again shortly.</span>';
              chatHeader.textContent = '💬 All Agents Busy';
            }
          } else {
            availabilityMsg.innerHTML = '<span class="error-message">Unexpected response format.</span>';
            chatHeader.textContent = '⚠️ Error';
          }

        } catch (error) {
          console.error('Error querying Salesforce:', error);
          availabilityMsg.innerHTML = `<span class="error-message">Connection error: ${error.message}</span>`;
          chatHeader.textContent = '⚠️ Connection Error';
        }
      }

      // Initialize on page load
      window.onload = function() {
        console.log('Page loaded, initializing...');
        initializeAuth();
        querySalesforceAccount();

        // Optional: Poll for availability updates every 30 seconds
        setInterval(querySalesforceAccount, 30000);
      };

    })();
  </script>
</apex:page>
