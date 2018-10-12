# Handling decision responses from Veriff


To automate the handling of responses from verifications, you will need to implement a webhook listener, according to the documentation at https://developers.veriff.me/#webhooks_decision_post
Once you have determined the URL of the listener, it is possible to configure it in the Veriff back office (Integrations / Notifications / Decision webhook url )
In case you want to test web hooks before the mobile app fully working, it is possible to hand generate verification sessions in the back office (Verifications / Generated tokens / Generate session and then following the URL for a web-based verification session, which will post the result to your webhook)


# Configuring the webhook endpoint

Go to Veriff's back office, Management -> Vendors -> Edit, and set 'Decision notification url' to the URL where your server will be listening for decision notifications from Veriff.  Veriff will post decision notifications of verification results to this URL.  Only HTTPS URLs are allowed.

If there is a network connectivity issue or technical issue with delivering the notification (any non-200 response code), Veriff will retry the notification multiple times for up to a week.  

The full description of the webhook format is at https://developers.veriff.me/#webhooks_decision_post


# What do the various verification responses mean
We give a positive conclusive decision (status approved, code 9001)  when the user has provided us with:
- photos and video uploaded to us
- the document is readable and matches the user's name as provided by the customer
- the user's portrait photo is recognizable
- the user on the portrait photo corresponds to the person on the document photo
- the document is valid (dates, etc)
A positive decision means that the person was verified. The verification process is complete. Accessing the KYC session URL again will tell the user that nothing more is to be done.

We give a negative conclusive decision (status declined, code 9102) when
- No physical document was used
- The document was unrecognizable
- The document was invalid (expired)
- The document was damaged
- The person did not match the document
 A negative decision means that the person has not been verified. The verification process is complete. Either it was a fraud case or some other severe reason that the person can not be verified. You should investigate the session further and read the "reason". If you decide to give the client another try you need to create a new session.

We give an inconclusive decision (status resubmission_requested, code 9103), when
- All necessary photos exist, but some data or the face are unrecognizable due to low quality
- insufficient lighting
- distracting factors
- there were mistakes during the verification process
- wrong type of document
- the person is not alone during verification
A resubmission requested decision means that the verification process is not completed. Something was missing from the user and they need to go through the flow once more. The same session URL can and should be used for this purpose.

