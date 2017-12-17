# Facebook Messenger Bot and Facebook Analytics
Training material of Facebook Messenger Bot and Facebook Analytics for [Facebook Masterclass Training](http://fbmasterclass4devs.id/). Facebook Masterclass Training is official workshop training provided by Facebook.

This repo is a revision of [prayudiutomo/Messenger-Bot-Ngrok-Tutorial](https://github.com/prayudiutomo/Messenger-Bot-Ngrok-Tutorial)

---

### WHAT IS NEEDED?

---

- NodeJS
- Facebook Page
- Facebook App
- Webhook Service
- Ngrok

---

### STEP I: BUILD THE SERVER

#### 1. Install NodeJS, NPM and Express

  a. Download and install **NodeJS** from official website: https://nodejs.org/.

  b. Check version.
  ```sh
  node -v
  ```

  c. Install latest **npm**
  ```sh
  npm install npm -g
  ```

  d. Create directory anywhere
  ```sh
  mkdir myBot
  cd myBot
  npm init
  ```

  e. Install Express is for the server, request is for sending out messages and body-parser is to process messages
  ```sh
  npm install express request body-parser --save
  ```


#### 2. Install and Run Ngrok

  a. Download and install **Ngrok** from official website: https://ngrok.com/

  b. Unzip from a terminal to any directory. On Windows, just double click ngrok.zip.
  ```sh
  unzip /path/to/ngrok.zip
  ```

  c. Start the **ngrok**
  ```sh
  ngrok http 5000
  ```


#### 3. Create file index.js

  ```javascript
  'use strict'

  const express = require('express')
  const bodyParser = require('body-parser')
  const request = require('request')
  const app = express()

  app.set('port', (process.env.PORT || 5000))

  // Process application/x-www-form-urlencoded
  app.use(bodyParser.urlencoded({extended: false}))

  // Process application/json
  app.use(bodyParser.json())

  // Index route
  app.get('/', function (req, res) {
    res.send('Hello world, I am a chat bot')
  })

  // for Facebook verification
  app.get('/webhook/', function (req, res) {
    if (req.query['hub.verify_token'] === 'make_indonesian_great_again') {
      res.send(req.query['hub.challenge'])
    }
    res.send('Error, wrong token')
  })

  // Spin up the server
  app.listen(app.get('port'), function() {
    console.log('running on port', app.get('port'))
  })
  ```

#### 4. Run server on index.js

```javascript
node index.js
```

---

### STEP II: SETUP FACEBOOK PAGE AND APP

#### 1. Create Facebook Page

  a. Click https://www.facebook.com/pages/create

#### 2. Create Facebook App

  a. Click https://developers.facebook.com/apps/

  b. Add Product **Facebook Messenger**

  c. Test connection
  ```sh
  curl -X POST "https://graph.facebook.com/v2.6/me/subscribed_apps?access_token=PAGE_ACCESS_TOKEN"
  ```

#### 3. Set up Webhook

  a. Click **Messenger - Settings**

  b. Click **Setup Webhooks**

  c. Add URL from **ngrok**
  ```sh
  https://blabla.ngrok.io
  ```

#### 4. Open a browser and verify that your webhook is reachable
```sh
  https://blabla.ngrok.io/webhook
  ```

---

### STEP III: SETUP THE BOT

#### 1. Add AccessToken
```javascript
const token = "<PAGE_ACCESS_TOKEN>"
```

#### 1. Add an API endpoint to index.js to process messages
```javascript
app.post('/webhook/', function (req, res) {
    let messaging_events = req.body.entry[0].messaging
    for (let i = 0; i < messaging_events.length; i++) {
	    let event = req.body.entry[0].messaging[i]
	    let sender = event.sender.id
	    if (event.message && event.message.text) {
		    let text = event.message.text
		    sendTextMessage(sender, "Text received, echo: " + text.substring(0, 200))
	    }
    }
    res.sendStatus(200)
})
```

#### 2. Create function to send text messages
```javascript
function sendTextMessage(sender, text) {
    let messageData = { text:text }
    request({
	    url: 'https://graph.facebook.com/v2.6/me/messages',
	    qs: {access_token:token},
	    method: 'POST',
		json: {
		    recipient: {id:sender},
			message: messageData,
		}
	}, function(error, response, body) {
		if (error) {
		    console.log('Error sending messages: ', error)
		} else if (response.body.error) {
		    console.log('Error: ', response.body.error)
	    }
    })
}
```

---

### STEP IV: AUTHENTICATION

#### 1. Improve Authentication

  a. Add crypto library
  ```javascript
  const crypto = require('crypto')
  ```

  b. Add APP Secret
  ```javascript
  const AppSecret = 'APP_YOUR_SECRET';
  ```

  c. Check first
  ```javascript
  app.use(bodyParser.json({verify: verifyRequestSignature}))
  ```

  d. Add function to verify
  ```javascript
  function verifyRequestSignature(req, res, buf){
    let signature = req.headers["x-hub-signature"];
    
    if(!signature){
      console.error('You dont have signature')
    } else {
      let element = signature.split('=')
      let method = element[0]
      let signatureHash = element[1]
      let expectedHash = crypto.createHmac('sha1', AppSecret).update(buf).digest('hex')

      console.log('signatureHash = ', signatureHash)
      console.log('expectedHash = ', expectedHash)
      if(signatureHash != expectedHash){
        console.error('signature invalid, send message to email or save as log')
      }
    }
  }
  ```

---

### STEP V: IMPROVE

#### 1. Improve code above!

```javascript
app.post('/webhook/', function (req, res) {
  let data = req.body
  if(data.object == 'page'){
    data.entry.forEach(function(pageEntry) {
      pageEntry.messaging.forEach(function(messagingEvent) {
        if(messagingEvent.message.text){
          sendTextMessage(messagingEvent.sender.id,messagingEvent.message.text);
        } else {
          sendTextMessage(messagingEvent.sender.id,'Service Belum Support Untuk Mendeteksi Hal ini');
        }
      }); 
    });
    res.sendStatus(200)
  }
})
```

#### 2. Improve Again!

```javascript
function sendTextMessage(sender, text) {
  let url = `https://graph.facebook.com/v2.6/${sender}?fields=first_name,last_name,profile_pic&access_token=${token}`;
  
  request(url, function (error, response, body) {
    if (!error && response.statusCode == 200) {
      let parseData = JSON.parse(body);
      let messageData = {
        text: `Hi ${parseData.first_name} ${parseData.last_name}, you send message : ${text}`
      }
      request({
        url: 'https://graph.facebook.com/v2.10/me/messages',
        qs: {
          access_token: token
        },
        method: 'POST',
        json: {
          recipient: {
            id: sender
          },
          message: messageData,
        }
      }, function (error, response, body) {
        if (error) {
          console.log('Error sending messages: ', error)
        } else if (response.body.error) {
          console.log('Error: ', response.body.error)
        }
      })
    }
  })
}
```

#### 3. Create Kerang Ajaib-like bot

```javascript
app.post('/webhook/', function (req, res) {
  let data = req.body
  if(data.object == 'page'){
    data.entry.forEach(function(pageEntry) {
      pageEntry.messaging.forEach(function(messagingEvent) {
        console.log(messagingEvent)
        if(messagingEvent.message.text.indexOf('?') > 0){
          var randInt =  Math.floor(Math.random() * (1 - 0 + 1)) + 0;
          var strPost = 'Ya';
          if(randInt === 0) strPost = 'Tidak';
          sendTextMessage(messagingEvent.sender.id, strPost);
        } else {
          sendTextMessage(messagingEvent.sender.id, 'Maaf saya hanya bisa menjawab pertanyaan yang dimulai dengan kata apakah dan diakhiri dengan tanda tanya. Contoh: Apakah saya jago ?');
        }
      }); 
    });
    res.sendStatus(200)
  }
})
```

#### 4. Set a Welcome Message

```ssh
curl -X POST -H "Content-Type: application/json" -d '{
  "greeting": [
    {
      "locale":"default",
      "text":"Hello!" 
    }, {
      "locale":"en_US",
      "text":"Hello you there"
    }, {
      "locale":"id_ID",
      "text":"Halo Apakabar ?"
    }
  ]
}' "https://graph.facebook.com/v2.6/me/messenger_profile?access_token=<PAGE_ACCESS_TOKEN>"
```

#### 4. Quick Replies

- Options button

```javascript
function sendOptionsButton(sender) {
  let messageData = {
    text: 'Kamu suka film apa',
    "quick_replies": [
       {
        "content_type":"text",
        "title":"Drama Romantis",
        "payload":"DRAMA"
      },
      {
        "content_type":"text",
        "title":"Aksi",
        "payload":"AKSI"
      },
      {
        "content_type":"text",
        "title":"Humor",
        "payload":"HUMOR"
      },
      {
        "content_type":"text",
        "title":"Lainnya",
        "payload":"LAINNYA"
      },
    ]
  }
  sendMessage(sender, messageData)
}
```

```javascript
function replyAnswerMovie(text) {
  if(text === 'DRAMA') {
    return 'Berarti anda suka AADC dong'
  } else if(text === 'AKSI') {
    return 'Film-film nya Bruce Willis asik tuh buat kamu'
  } else if(text === 'HUMOR') {
    return 'Warkop udah nonton belum ?'
  } else if(text === 'LAINNYA') {
    return 'Cintailah Film Indonesia'
  } else {
    return null
  }
}
```

```javascript
if(messagingEvent.message.quick_reply) {
  const replyMovie = replyAnswerMovie(messagingEvent.message.quick_reply.payload)
  console.log('replyMovie', replyMovie)
  if(replyMovie !== null) {
    sendTextMessage(messagingEvent.sender.id, replyMovie);
    return;
  }
}
```

```javascript
else if(messagingEvent.message.text.toLowerCase().indexOf('opsi') >= 0){
  sendOptionsButton(messagingEvent.sender.id)
} 
```

- location

```javascript
function sendLocationOption(sender) {
  let messageData = {
    text: 'Please Share your location',
    "quick_replies": [
      {
        "content_type":"location"
      },
    ]
  }
  sendMessage(sender, messageData)
}
```

```javascript
else if(messagingEvent.message.text.toLowerCase().indexOf('location') >= 0){
  sendLocationOption(messagingEvent.sender.id)
} 
```


## Excercise & Presentation (Let out your creativity - Create your own bot)
