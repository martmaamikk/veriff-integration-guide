# Generating sessions for KYC with Veriff


# Generating sessions manually


If you have managed to set up your account and logged in into our back office then you can generate a session:
- Click "Verification" on left menu
- Click "Generated tokens" from the opened menu
- Click "Generate general session" button on the top right
- Fill in the first name, last name and change the default language to "EN" (you can also change it during the flow). Just leave the SelfID checkbox choosed like it is already. 
- Click "Generate URL"
- Copy the URL into a different tab and you'll find yourself in the SelfID (verification without video interview) flow
- Finish the flow

You can see all the completed sessions from the menu “Dashboard”


# Generating sessions using JavaScript

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

