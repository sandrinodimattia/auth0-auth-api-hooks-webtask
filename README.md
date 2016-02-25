# Webhooks for Auth0's Authentication API with a Webtask

A webtask that allows you to define webhooks for Auth0's Authentication API. It will go through the audit logs and call a webhook for specific events. This can address use cases like:

> I want to be notified each time a user is blocked

> I want to be notified each time a login fails

There's also a webtask that exposes [webhooks for the Management API](https://github.com/sandrinodimattia/auth0-api-hooks-webtask).

## Configure Webtask

If you haven't configured Webtask on your machine run this first:

```
npm i -g wt-cli
wt init
```

> Requires at least node 0.10.40 - if you're running multiple version of node make sure to load the right version, e.g. "nvm use 0.10.40"

## Deployment

If you just want to run it once:

```
wt create https://raw.githubusercontent.com/sandrinodimattia/auth0-auth-api-hooks-webtask/master/task.js \
    --name auth0-api-hooks \
    --secret AUTH0_DOMAIN="YOUR_AUTH0_DOMAIN" \
    --secret AUTH0_GLOBAL_CLIENT_ID="YOUR_AUTH0_GLOBAL_CLIENT_ID" \
    --secret AUTH0_GLOBAL_CLIENT_SECRET="YOUR_AUTH0_GLOBAL_CLIENT_SECRET" \
    --secret LOG_LEVEL="1" \
    --secret LOG_TYPES="s,f" \
    --secret WEBHOOK_URL="http://my.webhook.url/something" \
    --secret WEBHOOK_CONCURRENT_CALLS="10"
```

If you want to run it on a schedule (run every 5 minutes for example):

```
wt cron schedule \
    --name auth0-api-hooks \
    --secret AUTH0_DOMAIN="YOUR_AUTH0_DOMAIN" \
    --secret AUTH0_GLOBAL_CLIENT_ID="YOUR_AUTH0_GLOBAL_CLIENT_ID" \
    --secret AUTH0_GLOBAL_CLIENT_SECRET="YOUR_AUTH0_GLOBAL_CLIENT_SECRET" \
    --secret LOG_LEVEL="1" \
    --secret LOG_TYPES="s,f" \
    --secret WEBHOOK_URL="http://my.webhook.url/something" \
    --secret WEBHOOK_CONCURRENT_CALLS="10" \
    --json \
    "*/5 * * * *" \
    https://raw.githubusercontent.com/sandrinodimattia/auth0-auth-api-hooks-webtask/master/task.js
```

> You can get your Global Client Id/Secret here: https://auth0.com/docs/api/v1

The following settings are optional:

 - `LOG_LEVEL`: This allows you to specify the log level of events that need to be sent.
 - `LOG_TYPES`: If you only want to send events with a specific type (eg: failed logins). This needs to be a comma separated list.
 - `WEBHOOK_CONCURRENT_CALLS`: Defaults to 5, these are the maximum concurrent calls that will be made to your web hook

## How it works

This webtask will process the Auth0 audit logs. Calls to the Authentication API are logged there and will be picked up by the webtask. These will then optionally be filtered will then be sent to your Webhook URL (POST).

Note that, if the URL we are sending the events to is offline or is returning anything that is not 200 OK (eg: Internal Server Error) processing will stop and the batch of logs will be retried in the next run. This means 2 things:

 - Logs are sent at least once, so make sure your webhook is idempotent

To test this you could easily setup an endpoint on `http://requestb.in/` and use that as a Webhook URL.

## Filters

The `LOG_LEVEL` can be set to (setting it to a value will also send logs of a higher value):

 - `1`: Debug messages
 - `2`: Info messages
 - `3`: Errors
 - `4`: Critical errors

The `LOG_TYPES` filter can be set to:

- `s`: Success Login (level: 1)
- `seacft`: Success Exchange (level: 1)
- `feacft`: Failed Exchange (level: 3)
- `f`: Failed Login (level: 3)
- `w`: Warnings During Login (level: 2)
- `du`: Deleted User (level: 1)
- `fu`: Failed Login (invalid email/username) (level: 3)
- `fp`: Failed Login (wrong password) (level: 3)
- `fc`: Failed by Connector (level: 3)
- `fco`: Failed by CORS (level: 3)
- `con`: Connector Online (level: 1)
- `coff`: Connector Offline (level: 3)
- `fcpro`: Failed Connector Provisioning (level: 4)
- `ss`: Success Signup (level: 1)
- `fs`: Failed Signup (level: 3)
- `cs`: Code Sent (level: 0)
- `cls`: Code/Link Sent (level: 0)
- `sv`: Success Verification Email (level: 0)
- `fv`: Failed Verification Email (level: 0)
- `scp`: Success Change Password (level: 1)
- `fcp`: Failed Change Password (level: 3)
- `sce`: Success Change Email (level: 1)
- `fce`: Failed Change Email (level: 3)
- `scu`: Success Change Username (level: 1)
- `fcu`: Failed Change Username (level: 3)
- `scpn`: Success Change Phone Number (level: 1)
- `fcpn`: Failed Change Phone Number (level: 3)
- `svr`: Success Verification Email Request (level: 0)
- `fvr`: Failed Verification Email Request (level: 3)
- `scpr`: Success Change Password Request (level: 0)
- `fcpr`: Failed Change Password Request (level: 3)
- `fn`: Failed Sending Notification (level: 3)
- `limit_wc`: Blocked Account (level: 4)
- `limit_ui`: Too Many Calls to /userinfo (level: 4)
- `api_limit`: Rate Limit On API (level: 4)
- `sdu`: Successful User Deletion (level: 1)
- `fdu`: Failed User Deletion (level: 3)

So for example, if I want to filter on a few events I would set the `LOG_TYPES` filter to: `sce,fce,scu,fcu`.

## Sample Payload

Here's an example of the payload that will be sent:

```
{
  date: '2016-02-25T13:42:08.791Z',
  type: 'f',
  description: 'Wrong email or password.',
  connection: 'My-Users',
  client_id: 'lIkP1Wn4qQPj56k9bE7fyMrbsaaHXd6c',
  client_name: 'Default App',
  ip: '11.22.33.44',
  user_agent: 'Chrome 48.0.2564 / Mac OS X 10.11.3',
  details:
   { error:
      { message: 'Wrong email or password.',
        oauthError: 'Wrong email or password.',
        type: 'invalid_user_password' },
     body:
      { client_id: 'lIkP1Wn4qQPj56k9bE7fyMrbsaaHXd6c',
        username: 'john@example.com',
        password: '*****',
        connection: 'My-Users'',
        grant_type: 'password',
        scope: 'openid',
        device: '' },
     qs: {},
     connection: 'My-Users' },
  user_id: '',
  user_name: 'Default App',
  strategy: 'auth0',
  strategy_type: 'database',
  _id: '49556539073893675610923042044589174982043486779166687234',
  isMobile: false
}
```

## Troubleshooting

To troubleshoot you can use the `wt logs` command to see the output of your Webtask in real time:

![](media/webtask-troubleshooting.png)
