<apex:page controller="ChatFlowController"
           showHeader="false"
           sidebar="false"
           standardStylesheets="false">

  <style>
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

    .chat-header.offline {
      display: none;
    }

    #availabilityMessage {
      padding: 14px;
      font-size: 14px;
      line-height: 1.6;
      text-align: left;
      background: #fff;
      color: #222;
    }

    .offline-label {
      display: inline-block;
      background: rgb(137, 211, 41);
      color: white;
      padding: 10px 16px;
      font-weight: bold;
      border-radius: 12px;
      margin-bottom: 12px;
      text-align: center;
      width: 100%;
    }

    .chat-hours {
      font-weight: bold;
      color: #000;
      display: block;
      margin: 6px 0 10px 0;
    }

    .submit-link-btn {
      display: inline-block;
      background: rgb(137, 211, 41);
      color: white !important;
      padding: 10px 18px;
      font-size: 14px;
      font-weight: bold;
      border-radius: 10px;
      margin-top: 10px;
      text-decoration: none;
      text-align: center;
    }

    .submit-link-btn:hover {
      background: rgb(110, 175, 33);
    }

    #launchChatBtn {
      background: rgb(137, 211, 41);
      color: white;
      border: none;
      padding: 14px 22px;
      font-size: 16px;
      font-weight: bold;
      border-radius: 12px;
      cursor: pointer;
      display: none;
      position: fixed;
      bottom: 20px;
      right: 20px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.25);
    }

    #launchChatBtn:hover {
      background: rgb(110, 175, 33);
      transform: translateY(-2px);
    }
  </style>

  <!-- CHAT CARD -->
  <div id="chat-widget">
    <div id="chat-header" class="chat-header">
      💬 Checking availability...
    </div>
    <p id="availabilityMessage">Please wait...</p>
  </div>

  <!-- CUSTOM LAUNCH BUTTON -->
  <button id="launchChatBtn"
          onclick="document.dispatchEvent(new Event('launchMIAWChatEvent'))">
    Chat with a Specialist
  </button>

  <!-- MIAW ROOT -->
  <div id="embedded-messaging-root"></div>

  <!-- EMBEDDED MESSAGING INIT -->
  <script type="text/javascript">
    function initEmbeddedMessaging() {
      try {
        embeddedservice_bootstrap.settings.language = 'en_US';
        embeddedservice_bootstrap.init(
          '00D9O000005Id3K',
          'MIDAS_US_CHAT_WEB',
          'https://bayeragmiidas--test.sandbox.my.site.com/ESWMIDASUSCHATWEB1736761433673',
          { scrt2URL: 'https://bayeragmiidas--test.sandbox.my.salesforce-scrt.com' }
        );
      } catch (err) {
        console.error('Embedded Messaging init error:', err);
      }
    }
  </script>

  <script type="text/javascript"
    src="https://bayeragmiidas--test.sandbox.my.site.com/ESWMIDASUSCHATWEB1736761433673/assets/js/bootstrap.min.js"
    onload="initEmbeddedMessaging()">
  </script>

  <!-- AVAILABILITY LOGIC -->
  <script>
    window.isWithinBH = null;
    window.availability = null;
    window.numberOfAgents = null;

    async function querySalesforceAccount() {

      const msg = document.getElementById('availabilityMessage');
      const btn = document.getElementById('launchChatBtn');
      const header = document.getElementById('chat-header');
      const card = document.getElementById('chat-widget');

      try {
        const response = await fetch(
          '/services/apexrest/CycleExample',
          { method: 'GET' }
        );

        if (!response.ok) {
          throw new Error('Apex REST error');
        }

        const obj = await response.json();

        window.isWithinBH = obj.isWithinBH;
        window.availability = obj.Availability;
        window.numberOfAgents = obj.numberofAgent;

        if (window.numberOfAgents > 0 && window.isWithinBH) {
          header.style.display = "none";
          card.style.display = "none";
          btn.style.display = "block";
        } else {
          card.style.display = "block";
          btn.style.display = "none";
          header.classList.add("offline");

          if (!window.isWithinBH) {
            msg.innerHTML =
              "<span class='offline-label'>🕒 Outside Business Hours</span>" +
              "<div>Hello, our specialists are currently unavailable.<br/>" +
              "<span class='chat-hours'>Monday – Friday<br/>8:30 AM – 8:00 PM EST</span>" +
              "Please submit your question:<br/>" +
              "<a class='submit-link-btn' href='https://askmed.bayer.com/submit-question' target='_blank'>Submit Question</a></div>";
          } else {
            msg.innerHTML =
              "<span class='offline-label'>💬 Agents are Offline</span>" +
              "<div>Please try again later or submit your question:<br/>" +
              "<a class='submit-link-btn' href='https://askmed.bayer.com/submit-question' target='_blank'>Submit Question</a></div>";
          }
        }

      } catch (e) {
        console.error(e);
        header.innerText = "⚠️ Error";
        msg.innerText = "There was a problem checking availability.";
        btn.style.display = "none";
      }
    }

    querySalesforceAccount();

    /* MIAW EVENTS */
    window.addEventListener('onEmbeddedMessagingReady', function () {
      embeddedservice_bootstrap.settings.hideChatButtonOnLoad = true;

      document.addEventListener('launchMIAWChatEvent', function () {
        embeddedservice_bootstrap.utilAPI.launchChat();
      });

      window.addEventListener(
        'onEmbeddedMessagingWindowMinimized',
        function () {
          const btn = document.getElementById('launchChatBtn');
          if (btn) btn.style.display = "none";
        }
      );
    });
  </script>

</apex:page>
