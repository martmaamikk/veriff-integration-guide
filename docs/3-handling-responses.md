# Handling decision responses from Veriff


To automate the handling of responses from verifications, you will need to implement a webhook listener, according to the documentation at https://developers.veriff.me/#webhooks_decision_post
Once you have determined the URL of the listener, it is possible to configure it in the Veriff back office (Integrations / Notifications / Decision webhook url )
In case you want to test web hooks before the mobile app fully working, it is possible to hand generate verification sessions in the back office (Verifications / Generated tokens / Generate session and then following the URL for a web-based verification session, which will post the result to your webhook)


## Configuring the webhook endpoint

Go to Veriff's back office, Management -> Vendors -> Edit, and set 'Decision notification url' to the URL where your server will be listening for decision notifications from Veriff.  Veriff will post decision notifications of verification results to this URL.  Only HTTPS URLs are allowed.

If there is a network connectivity issue or technical issue with delivering the notification (any non-200 response code), Veriff will retry the notification multiple times for up to a week.  

The full description of the webhook format is at https://developers.veriff.me/#webhooks_decision_post

## Recognizing your customer

When your server receives a decision notification from Veriff, you have to figure out, which customer is it about.

There are two ways to do this:
  1) using the Veriff session ID, or
  2) using your own customer ID.

The easiest way is to track the session ID provided by Veriff during session creation.  All future event and decision notifications refer to the session ID.  The session ID is unique for each session, and it can be used to look up sessions in the administrative interface at https://office.veriff.me

The other way is to provide Veriff with your internal customer ID, or some other key that uniquely identifies your customer.  You can store your identifier in the vendorData element as a string, and we will send it back to you in webhook notifications.  Please bear in mind that it is technically possible for one customer to be associated with multiple verification sessions, and this could potentially create ambiguous situations in code, if you are only recognizing customers by your own identifier, and not Veriff's session ID.

## Handling security

It is important to check that the webhook responses do indeed originate from Veriff.

You can secure the webhook listener URL in three ways:
- have a good secure SSL server for your webhook listener (Veriff will call only to HTTPS URLs, to servers with a publicly verifiable certificate)
- check the X-AUTH-CLIENT and X-SIGNATURE headers on the decision webhook (the signature is calculated using the API secret that only you and Veriff know)
- finally, if you are really suspicious, you may restrict your webhook listener to only accept calls from the Veriff IP range (please ask your Veriff integration onboarding specialist for those details)


When Veriff calls your webhook endpoint, we use the same logic of X-SIGNATURE generation on our calls to you, as on your calls to us.  So, when we call your endpoint for any notification URL, we set the X-AUTH-CLIENT http header to your API key, and we set the X-SIGNATURE header to the hex encoded sha256 digest of the request body and the secret.

When you accept a webhook call from us, you need to proceed as follows:
1) compare the X-AUTH-CLIENT header to your api key, if different -> **fail** with error 
2) access the http request body (before it is parsed)
3) calculate the sha256 digest of the request body string + api secret
4) compare the hex encoded digest with the X-SIGNATURE header, if different -> **fail** with error
5) only now you should parse the request body JSON object

The calculation for X-SIGNATURE is following the same algorithm as for session generation.


## Different webhook calls

The difference between the URLs is as follows:
- the Decision Webhook Url is where we post decisions about verifications (approved, declined, resubmission_required, expired, abandoned, etc)
- the Event Webhook Url is where we post events during the lifecycle of a verification (started, submitted, etc)
- finally, there is a Callback URL, which is actually the redirect URL for the user on the web site at the end of KYC


## Meaning of the various verification responses

Verification status is one of 
- approved
- resubmission_requested
- declined
- expired
- abandoned
  
Verification response code is one of 9001, 9102, 9103, 9104, 9151, 9161

Explanation of the meaning of the response codes: 
- **9001** : Positive: Person was verified. The verification process is complete. Accessing the sessionURL again will show the client that nothing is to be done here.
- **9102** : Negative: Person has not been verified. The verification process is complete. Either it was a fraud case or some other severe reason that the person can not be verified. You should investigate the session further and read the "reason". If you decide to give the client another try you need to create a new session.
- **9103** : Resubmitted: Resubmission has been requested. The verification process is not completed. Something was missing from the client and she or he needs to go through the flow once more. The same sessionURL can and should be used for this purpose.
- **9104** : Negative: Verification has been expired. The verification process is complete. After 7 days the session get's expired. If the client started the verification process we reply "abandoned" here, otherwise if the client never arrived in our environment the status will be "expired"
- **9151** : Intermediate Positive: SelfID was successful - this code is only send if the configuration flag is set
- **9161** : Intermediate Positive: Video Call was successful - this code is only send if the configuration flag is set'

### Lifecycle of the verification session

Responses 9001, 9102, and 9104 are *conclusive* responses.  The session is closed and the URL is *not* available for the end user.

Response 9103 is an *inconclusive* response.  The session remains open until you receive one of the above conclusive responses.  The session is re-opened for the end user, accepting a new attempt.

Responses 9151 and 9161 are *intermediate* responses.  The session is being processed, it is not open for the end user, and you will soon receive one of the above conclusive or inconclusive responses.

### Preconditions for approval decisions

We give a positive conclusive decision (status approved, code 9001)  when the user has provided us with:
- photos and video uploaded to us
- the document is readable and matches the user's name as provided by the customer
- the user's portrait photo is recognizable
- the user on the portrait photo corresponds to the person on the document photo
- the document is valid (dates, etc)
A positive decision means that the person was verified. The verification process is complete. Accessing the KYC session URL again will tell the user that nothing more is to be done.

### Reasons for negative conclusive decisions

We give a negative conclusive decision (status declined, code 9102) when

- Document type is not supported.
- The name entered and the name on the document do not match.
- A physical ID document was not used.
- Document is damaged.
- Document has signs of tampering.
- Video has signs of tampering.
- Document is expired.
- No matching document specimen available.
- The person showing the document does not look the same as on the document photo.
- The person displays signs of being intoxicated.

 A negative decision means that the person has not been verified. The verification process is complete. Either it was a fraud case or some other severe reason that the person can not be verified. You should investigate the session further and read the "reason". If you decide to give the client another try you need to create a new session.

### Reasons for inconclusive decisions

We give an inconclusive decision (status resubmission_requested, code 9103), when

- Photos necessary for identification are missing.
- The face is not clearly visible on the portrait photo.
- The document is not fully in frame.
- Document data is unreadable due to bad quality.
- Document data is unreadable due to being partially covered.
- The process of showing the document was not completed independently.

An inconclusive (resubmission requested) decision means that the verification process is not completed. Something was missing from the user and they need to go through the flow once more. The same session URL can and should be used for this purpose.

## Meaning of the various event codes

The event webhook informs you about the progress of the verification.  However, it does not inform you of the decision.    

The event code can be one of:
- '**7001** : Started
- '**7002** : Submitted
- '**7003** : Tier 1 approved


Example

    {
      "id": "f04bdb47-d3be-4b28-b028-a652feb060b5",
      "feature": "selfid",
      "code": 7003,
      "action": "tier_1_approved"
    }

## Storing verification media

You can automate the download of photo and video files associated with a completed verification.

First, query for a list of files using a GET request to

    /sessions/{sessionId}/media

The response will be a list of videos and images as described at https://developers.veriff.me/#sessions__sessionid__media_get

Second, you can then download each individual media file, using the media ID returned in the first step, with a GET request to

    /media/{mediaId}


The description of the request and response is at https://developers.veriff.me/#media__mediaid__get
