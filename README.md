<script type="text/javascript">
  // Replace these values with the actual values from your Salesforce connected app
  const consumerKey = '3MVG9CG8tifMyyO1fMXNWesnELP55WB36HsKrY_Ry6Z10rbiscxArlyFyiNbJnupsMv.ot4VpmXHUdQLupteW';  // Consumer Key
  const consumerSecret = '41BA55FEA12C182A2F43F1544ED0632A300DC9C25B2C6A3C411F3C67E4416A38';  // Consumer Secret
  const redirectUri = 'https://www.getpostman.com/oauth2/callback';  // Callback URL (for testing in Postman)

  // OAuth Authorization URL
  const authUrl = `https://bayeragmiidas--test.sandbox.my.salesforce.com/services/oauth2/authorize?response_type=code&client_id=3MVG9CG8tifMyyO1fMXNWesnELP55WB36HsKrY_Ry6Z10rbiscxArlyFyiNbJnupsMv.ot4VpmXHUdQLupteW&redirect_uri=http://localhost`;

  // Function to initiate the OAuth process (step 1 - Authorization)
  function getAuthorizationCode() {
    window.location.href = authUrl;  // Redirect to Salesforce OAuth authorization page
  }

  // Function to exchange Authorization Code for Access Token and Refresh Token
  async function getAccessToken(authorizationCode) {
    const tokenEndpoint = 'https://bayeragmiidas--test.sandbox.my.salesforce.com/services/oauth2/token';

    const body = new URLSearchParams({
      'grant_type': 'authorization_code',
      'code': authorizationCode,
      'client_id': consumerKey,
      'client_secret': consumerSecret,
      'redirect_uri': redirectUri
    });

    const response = await fetch(tokenEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: body.toString()
    });

    const data = await response.json();
    if (response.ok) {
      return data;  // Returns access_token and refresh_token
    } else {
      throw new Error('Failed to obtain access token: ' + data.error_description);
    }
  }

  // Refresh token logic (to obtain a new access token using refresh token)
  async function refreshAccessToken(refreshToken) {
    const refreshEndpoint = 'https://bayeragmiidas--test.sandbox.my.salesforce.com/services/oauth2/token';

    const body = new URLSearchParams({
      'grant_type': 'refresh_token',
      'client_id': consumerKey,
      'client_secret': consumerSecret,
      'refresh_token': refreshToken
    });

    const response = await fetch(refreshEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: body.toString()
    });

    const data = await response.json();
    if (response.ok) {
      return data.access_token;  // New access token
    } else {
      throw new Error('Failed to refresh access token: ' + data.error_description);
    }
  }

  // Function to query Salesforce API with access token
  async function querySalesforceAccount() {
    const refreshToken = '5Aep861pw2VNBY3IWYyH3ybCjG64bbZTO4o9DUVi3s6VCellJnFOOOsQ8nhAh5V1NXBEAmkfpu_ncGXHPjU_JYd';  // Replace with your actual refresh token
    let accessToken = '00D9O000005Id3K!AQEAQBt3Vk7uzx3b3q7DZPtwLMhhT3cPGwDQ..6zBkan9zqHzLYOotWoFHYq9APpco5Vmy43CDo3xvVYlRbN4pYrUgZ.xD6o';  // Replace with your current access token
    try {
      // If the access token is expired, refresh it
      if (isTokenExpired(accessToken)) {
        accessToken = await refreshAccessToken(refreshToken);
      }

      const endpoint = 'https://bayeragmiidas--test.sandbox.my.salesforce.com/services/apexrest/CycleExample';
      const response = await fetch(endpoint, {
        method: 'GET',
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        }
      });

      if (!response.ok) {
        throw new Error('Error querying Salesforce');
      }

      const data = await response.json();
      console.log('Query Result:', data);

      // Handle the response
      if (data) {
        const obj = JSON.parse(data);
        window.isWithinBH = obj.isWithinBH;
        window.availability = obj.Availability;
        window.numberOfAgents = obj.numberofAgent;

        if (window.numberOfAgents > 0 && window.isWithinBH) {
          document.getElementById('launchChatBtn').style.display = 'block';
        } else {
          document.getElementById('launchChatBtn').style.display = 'none';
        }
      }
    } catch (error) {
      console.error('Error:', error);
      document.getElementById('availabilityMessage').innerText = 'Error querying Salesforce.';
    }
  }

  // Function to check if the token is expired
  function isTokenExpired(accessToken) {
    const payload = JSON.parse(atob(accessToken.split('.')[1]));  // Decode JWT token
    const expirationTime = payload.exp * 1000;  // Expiration time in milliseconds
    return Date.now() > expirationTime;
  }

  // Start OAuth and get access token
  window.onload = function () {
    querySalesforceAccount();
  };
</script>
