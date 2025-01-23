# IBM-Watson-Implementing-a-Facebook-Messenger-Chatbot-Powered-by-Watson-Conversation-using-ngrok
Creating a Facebook Messenger chatbot powered by IBM Watson Assistant (previously known as Watson Conversation) involves several steps. You need to set up a chatbot using Watson Assistant, implement a server to handle incoming messages, and use ngrok to expose your local server to the public internet for testing. ngrok acts as a tunnel between your local development environment and the outside world, allowing you to test the bot with Facebook Messenger.
Prerequisites:

    IBM Cloud Account:
        Create a Watson Assistant instance on IBM Cloud.
        Get your API key and Assistant ID from the Watson Assistant service.

    Facebook Developer Account:
        Create a Facebook app and set up Facebook Messenger.
        Get your Facebook Page Access Token and Verify Token from Facebook Developer Console.

    ngrok:
        Install ngrok to expose your local development server to the internet. You can download it from ngrok's website.

    Server Setup:
        We'll use Node.js to build the server and integrate the chatbot with Facebook Messenger and Watson Assistant.

Steps to Implement Facebook Messenger Chatbot Powered by Watson Assistant:
Step 1: Set up the Watson Assistant (formerly Watson Conversation)

    Create a Watson Assistant instance on IBM Cloud.
    Create an Assistant in Watson Assistant.
        Train the assistant by defining intents, entities, and dialog flows.
        You can create intents like greeting, goodbye, or specific tasks like order_food.
    Obtain the Assistant ID, API Key, and URL for your Watson Assistant instance.

Step 2: Set up the Facebook Messenger Chatbot

    Create a Facebook App:
        Go to the Facebook Developer Console, and create a new app.
        Add the Messenger product to your app.
        Create a Facebook Page and link it to your app.

    Get the Page Access Token:
        Under the Messenger product settings, generate a Page Access Token.
        You will need this token to send and receive messages from your Facebook Page.

    Generate a Verify Token:
        This is a token that Facebook uses to verify your webhook endpoint during the subscription process.
        The Verify Token is arbitrary, so you can set any string (e.g., my_verify_token).

    Subscribe to the Page:
        Set up your webhook URL to point to the endpoint where you will receive and respond to messages.
        Subscribe to the Facebook page events (messages, messaging_postbacks).

Step 3: Set up the Server with Node.js

You’ll need to create a simple Node.js server to handle webhook requests from Facebook Messenger and forward messages to Watson Assistant.
Install the required packages:

npm init -y
npm install express body-parser request ngrok ibm-watson

Example Code for Facebook Messenger Chatbot (Node.js):

const express = require('express');
const bodyParser = require('body-parser');
const request = require('request');
const { IamAuthenticator } = require('ibm-watson/auth');
const { AssistantV2 } = require('ibm-watson/assistant/v2');
const ngrok = require('ngrok');

// Facebook Messenger Token & Watson Assistant Credentials
const PAGE_ACCESS_TOKEN = 'YOUR_FACEBOOK_PAGE_ACCESS_TOKEN';
const VERIFY_TOKEN = 'YOUR_FACEBOOK_VERIFY_TOKEN';
const ASSISTANT_API_KEY = 'YOUR_WATSON_API_KEY';
const ASSISTANT_ID = 'YOUR_WATSON_ASSISTANT_ID';
const VERSION = '2021-06-14';

// Initialize Watson Assistant
const assistant = new AssistantV2({
  version: VERSION,
  authenticator: new IamAuthenticator({
    apikey: ASSISTANT_API_KEY
  }),
  serviceUrl: 'YOUR_WATSON_ASSISTANT_URL'
});

const app = express();
const port = process.env.PORT || 3000;

// Middleware
app.use(bodyParser.json());

// Endpoint to validate webhook for Facebook
app.get('/webhook', (req, res) => {
  const mode = req.query['hub.mode'];
  const token = req.query['hub.verify_token'];
  const challenge = req.query['hub.challenge'];

  // Check if the token matches
  if (mode && token === VERIFY_TOKEN) {
    res.status(200).send(challenge);
  } else {
    res.status(403).send('Error, wrong token');
  }
});

// Handle incoming messages from Facebook Messenger
app.post('/webhook', (req, res) => {
  const messaging_events = req.body.entry[0].messaging;

  // Loop through each messaging event
  for (let i = 0; i < messaging_events.length; i++) {
    const event = messaging_events[i];
    const sender = event.sender.id;

    // Check if the message is from a user
    if (event.message && event.message.text) {
      const userMessage = event.message.text;

      // Call Watson Assistant with the user's message
      sendMessageToWatsonAssistant(userMessage, sender);
    }
  }
  
  res.status(200).send('EVENT_RECEIVED');
});

// Send message to Watson Assistant
function sendMessageToWatsonAssistant(message, sender) {
  assistant.message({
    assistantId: ASSISTANT_ID,
    sessionId: 'YOUR_SESSION_ID', // You can generate or maintain a session ID per user
    input: { text: message }
  })
  .then(response => {
    const watsonResponse = response.result.output.text[0];
    sendTextMessage(sender, watsonResponse);
  })
  .catch(err => {
    console.error(err);
  });
}

// Send a text message to Facebook Messenger
function sendTextMessage(sender, text) {
  const messageData = {
    recipient: {
      id: sender
    },
    message: {
      text: text
    }
  };

  // Send the message to Facebook Messenger
  request({
    url: 'https://graph.facebook.com/v11.0/me/messages',
    qs: { access_token: PAGE_ACCESS_TOKEN },
    method: 'POST',
    json: messageData
  }, (error, response, body) => {
    if (error) {
      console.log('Error sending message: ', error);
    } else {
      console.log('Message sent: ', body);
    }
  });
}

// Start server
app.listen(port, () => {
  console.log(`Server is running on port ${port}`);

  // Expose localhost server using ngrok
  ngrok.connect(port).then(url => {
    console.log(`Ngrok tunnel opened at ${url}`);
  });
});

Explanation of the Code:

    Webhook Verification (/webhook):
        Facebook requires you to verify the webhook endpoint by checking the hub.mode, hub.verify_token, and hub.challenge parameters.
        The VERIFY_TOKEN is compared to ensure that the request is legitimate.

    Handling Incoming Messages:
        The /webhook endpoint listens for incoming POST requests when a user sends a message on Facebook Messenger.
        It checks the message.text of the event, and sends the message to Watson Assistant via the sendMessageToWatsonAssistant function.

    Sending Message to Watson Assistant:
        The assistant.message() method sends the user input to Watson Assistant for processing.
        Watson Assistant responds with the appropriate text, which is then sent back to the user on Facebook Messenger.

    Sending Messages Back to Facebook Messenger:
        The sendTextMessage() function sends the response back to the user on Facebook Messenger using the Facebook Graph API.

    Running ngrok:
        ngrok.connect(port) opens a public URL (tunnel) to your local server, making it accessible from the internet.
        You'll use this ngrok URL as the webhook URL for Facebook Messenger.

Step 4: Expose Local Server Using ngrok

Run the server:

node server.js

Then, ngrok will expose your local server to the internet:

ngrok http 3000

This will give you a public URL (e.g., https://abcd1234.ngrok.io). Copy this URL and set it as the Webhook URL in the Facebook Messenger settings.
Step 5: Connect Facebook Messenger to the Webhook

    In your Facebook Developer Console, go to Messenger settings, and paste the ngrok URL (https://abcd1234.ngrok.io/webhook) into the Webhook URL field.
    Set the Verify Token to the value you defined earlier (my_verify_token).
    Facebook will verify your webhook, and you should be good to go!

Step 6: Test the Chatbot

Send a message to your Facebook page, and the chatbot should respond with the output generated by Watson Assistant.
Conclusion

This implementation connects IBM Watson Assistant to Facebook Messenger using a Node.js server, with ngrok exposing your local server for testing. The server handles incoming messages from Facebook Messenger, sends them to Watson Assistant for processing, and then sends the response back to the user.

To deploy this bot to a production environment, you would typically host the server on a platform like Heroku, AWS, or IBM Cloud instead of using ngrok.
