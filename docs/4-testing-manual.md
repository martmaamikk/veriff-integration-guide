# Testing Manual for your Veriff Integration

Work in progress


## Testing KYC sessions

- Best case - new session link generated
- Session data is saved with your customer record
- Sessions with odd (exotic) names can be started
- Sessions with extra info, i.e. document details or vendor data
- In case of disrupted session (browser close, user logout, etc), user should be directed back to earlier session
- At end of KYC, callback URL redirects back to the correct place in your app

## Testing security

- Decision web hook with wrong API key should not be accepted
- Decision web hook with mismatched X-SIGNATURE should not be accepted
- Decision with invalid JSON should not break or crash your server


## Testing responses

Each type of response should be accepted:
- Approved
- Declined
- Resubmission Requested
- Expired
- Abandoned

## Testing process in your app

You should test the handling of best cases as well as edge cases

- approved users handled or notified as appropriate
- declined users handled as appropriate, your office notified if necessary
- in case of resubmission request, user is invited back to try KYC again
- in case of expired or abandoned session (after 7 days), user is not offered to continue from same session


## Mobile testing

- to do

