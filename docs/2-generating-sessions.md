# Generating sessions for KYC with Veriff


## Generating sessions manually


If you have managed to set up your account and logged in into our back office then you can generate a session:
- Click "Verification" on left menu
- Click "Generated tokens" from the opened menu
- Click "Generate general session" button on the top right
- Fill in the first name, last name and change the default language to "EN" (you can also change it during the flow). Just leave the SelfID checkbox choosed like it is already. 
- Click "Generate URL"
- Copy the URL into a different tab and you'll find yourself in the SelfID (verification without video interview) flow
- Finish the flow

You can see all the completed sessions from the menu “Dashboard”



## Generating sessions in code

The simplest way to get a KYC link for your user on the web, is to make a simple JSON object containing the user's name and the redirect(callback) URL to which the user will be sent after they complete KYC.
Then use HTTP POST to send the object to https://api.veriff.me/v1/sessions, with Content-Type application/json and the X-AUTH-CLIENT header containing your API Key.

    {
        "verification": {
            "callback": "https://this.is.your.redirect.url",
            "person": {
                "firstName": "John",
                "lastName": "Smith"
            },
            "lang": "en",
            "features": ["selfid"],
            "timestamp": "2016-05-19T08:30:25.597Z"
        }
    }

In response, we give you a JSON  session ID and a URL, as follows:

    {
        "status": "success",
        "verification":{
            "id":"f04bdb47-d3be-4b28-b028-............",
            "url": "https://magic.veriff.me/v/sample-url.................",
            "sessionToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJh............",
            "baseUrl": "https://magic.veriff.me"
        }
    }


The URL of the KYC session is where you should redirect your user in your web front end.

The session ID should be saved on your end - you can tie back the webhook responses to your customer record that way.

Once the end user completes the KYC, they will be redirected back to the callback URL.
If you do not specify it for each session call, then there is a default configured in your account settings. This "callback" could be to a "thank you" or waiting page on your side.

To be clear, the callback does not contain any decision or verification information yet.  After the KYC process completes in the background, we notify your webhook endpoint (also configured in your account settings), or if you have not automated decisions yet, then you can look up the result by logging in to https://office.veriff.me and checking the Dashboard.


The full API for starting a KYC session with Veriff is documented here in the API: https://developers.veriff.me/#sessions_post


## Generating the X-SIGNATURE header

The X-SIGNATURE header guarantees to us, that API calls are initiated by you.  It is based on the shared secret (API secret) that is known only by you and by Veriff.

The X-SIGNATURE is a SHA-256 hex encoded hash of the concatenation of the request body and your API secret.

To generate the X-SIGNATURE header, please:
- serialize (stringify) the request (this should be your HTTP request body)
- in a temporary buffer, concatenate the request body and your API secret
- calculate a SHA 256 hash
- hex encode the hash

For example, if the request object is 

    {
        "verification": {
            "person": {
                "firstName": "John",
                "lastName": "Smith"
            },
            "features": ["selfid"],
            "timestamp": "2018-10-15T08:30:25.597Z"
        }
    }

Next, the serialized JSON payload of the above object will look something like the following string

    '{"verification":{"person":{"firstName":"John","lastName":"Smith"},"features":["selfid"],"timestamp":"2018-10-15T08:30:25.597Z"}}'

Then, you will need to concatenate the text of the serialized payload and the text of the API secret, so it will look something like

    '{"verification":{"person":{"firstName":"Joh...........}}aeebd8dc-f646-4bc2.....etc'

Now you will need to calculate a SHA256 hash and hex encode it, as the description of that calculation is in the API spec at https://developers.veriff.me/#sessions_post

This process depends on the language you use.  


### Using JavaScript / ECMA 

    const payload = JSON.stringify(verification);
    const signature = crypto.createHash('sha256');
    signature.update(new Buffer(payload, 'utf8'));
    signature.update(new Buffer(secret, 'utf8'));
    return signature.digest('hex');

### Using C# / .Net

    string hashString;
    using (var sha256 = SHA256Managed.Create())
    {
        var hash = sha256.ComputeHash(Encoding.Default.GetBytes(payloadPlusSecret));
        hashString = ToHex(hash, false);
    }

<!--
### Using Ruby
### Using PHP
### Using Java

## Postman samples
-->


## Using Veriff in an iFrame

You can use Veriff in an iFrame.  It should fit in a 1000px x 650px frame.  This is an example for creating the iframe with the session URL:

    var vf = document.createElement('iframe');
    vf.src = response.verification.url; /* URL here */
    vf.allow = 'microphone; camera';
    vf.width = 1000;
    vf.height = 650;
    document.body.appendChild(iframe);

At the end of the verification, Veriff will send an event to the parent window using JavaScript postMessage.  You could use something as follows to catch the event and proceed:

    /* addEventListener support for IE8 */
    function bindEvent(element, eventName, eventHandler) {
      if (element.addEventListener) {
        element.addEventListener(eventName, eventHandler, false);
      } else if (element.attachEvent) {
        element.attachEvent('on' + eventName, eventHandler);
      }
    }

    /* Listen to message from child window */
    bindEvent(window, 'message', function (e) {
      if (e.data && e.data.status && e.data.status === 'finished') {
        /* do whatever */
        document.getElementById('iframe').remove();
      }
    });


## API upload

There are a few scenarios where you may not wish to use Veriff's native SDK or Veriff's web interface to verify your customers.  For example:
- you wish to completely implement your own front-end
- you wish to do an offline bulk audit of previously verified customers

In those cases, you can do the whole process using our API, according to the documentation, and not show any Veriff front end to your customers, or not expect customers to be present for the verification.

Here are the steps you should do to use the API for upload.

### Create a new verification session

Create a new verification session using POST request to ‘/sessions’

The goal here is to create a new object (a verification session) that contains the one verification (referred to as 'verification', nested inside the session object in the response).

If you wish to restrict the accepted document to be the one you have on file, you can also send the document type, country, and number along with the session.  Document type can be one of [‘PASSPORT’, ‘ID_CARD’, ‘RESIDENCE_PERMIT’, ‘DRIVERS_LICENSE'].

Documentation: https://developers.veriff.me/#sessions_post

### Send photos

Send photos (face, document front, document back, etc) by uploading all the pictures one by one using POST request to ‘/media’

The goal here is to upload the required photos and associate them with the verification created in step 1.

Documentation: https://developers.veriff.me/#sessions__sessionid__media_post

### Submit session for review

Submit the verification session using PATCH request to ‘/sessions/:id’

Once all the photos are uploaded, you would then update (PATCH) the verification to mark it into 'submitted' status. This marks the completion of the verification. This step requires all the photos to be submitted prior to triggering this.

Documentation: https://developers.veriff.me/#sessions__sessionid__patch

After these three steps, you've done your part, and the verification will then be taken care of by us.

### Wait for webhook response

Veriff sends you a Decision event via Webhook using POST request.  You will have to implement the listener.

This hook will be sent back asynchronously, and it contains more data, including the verified identity information.

Documentation: https://developers.veriff.me/#webhooks_decision_post

The web hook is sent to the URL that is configurable from the Back office under Management › Vendor › Integrations.

More info and code samples are in our public github repo at https://github.com/Veriff/js-integration-demo
