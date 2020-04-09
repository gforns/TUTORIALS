# Flask-OIDC Authentication with AWS Cognito UI


In this case we used flask-oidc.

https://flask-oidc.readthedocs.io/

To integrate Cognito UI directly to Flask.



Amazon Cognito hosts an authentication server that allows you to add sign-up and sign-in webpages to your app. Examples on Web are only in Javascript.  

https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-integration.html

https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-userpools-server-contract-reference.html

https://aws.amazon.com/blogs/mobile/understanding-amazon-cognito-user-pool-oauth-2-0-grants/



Here integratino is done directly in the Flask using this 3 modules:

    pip3 install flask
    pip3 install pyOpenSSL
    pip3 install flask-oidc


We need to make sure that in aws we set:

    Callback URLS:   https://127.0.0.1:5000/oidc_callback
    Sign Out URLS:   https://127.0.0.1:5000



Flask-oidc configuration is done basically in a file called client_secrets.json.  The format needs to be like this:

```json
{
    "web": {
        "auth_uri": "{{AWS_DOMAIN_PREFIX}}/oauth2/authorize",
        "client_id": "{{AWS_APP_CLIENT_ID}}",
        "client_secret": "{{AWS_APP_CLIENT_SECRET}}",
        "redirect_uris": [
            "https://127.0.0.1:5000/oidc_callback"
        ],
        "token_uri":  "{{AWS_DOMAIN_PREFIX}}/oauth2/token",
        "userinfo_uri": "{{AWS_DOMAIN_PREFIX}}/oauth2/userInfo",
        "issuer": "https://cognito-idp.eu-west-1.amazonaws.com/{{AWS_POOL_ID}}"
    }
}
```


Then we can initialize our app:

```python

from flask import Flask, render_template, redirect
from flask_oidc import OpenIDConnect

app = Flask(__name__)

app.config.update({
    'DEBUG': True,
    'TESTING': True,
    'SECRET_KEY' : 'very_secret_key',
    'OIDC_ID_TOKEN_COOKIE_SECURE' : False,
    'OIDC_REQUIRE_VERIFIED_EMAIL': True,
    'OIDC_ID_TOKEN_COOKIE_TTL': 28800,
    'OIDC_CLIENT_SECRETS': "client_secrets.json",    #  In client_secrets.json is defined all URLs for Cognito UI
    'COGNITO_UI_BASE_URL': "{{AWS_DOMAIN_PREFIX}}",  # USED For logout
    'COGNITO_UI_CLIENT_ID': "{{AWS_APP_CLIENT_ID}}",  # USED For logout
    'COGNITO_UI_LOGOUT_REDIRECT': "https://127.0.0.1:5000/"   # USED For logout
})


oidc = OpenIDConnect(app)


# Endpoint / is public
@app.route('/')
def root_index():
    return "Public Endpoint. No autentication is needed"


if __name__ == "__main__":
    # AWS Cognito UI only  works with SSL (pyOpenSSL is needed) 
    app.run(ssl_context="adhoc")

``` 



For Private Endoints we can use the @oidc.require_login 


```py

# Enpoint /hello is private.  "require_login"  is needed to protect it. 
# Require_login will forward us to Cognito UI in case there is no oidc session created
@app.route('/hello')
@oidc.require_login
def index():
    # We can get email provieded on the authentication
    # Our app has to control permisions
    if has_permissions(oidc.user_getfield('email')):
        return "Private Endpoint. Logged in as: " + oidc.user_getfield('email') 
    else:
        return "User has no permision to get the resource"


# Helper function to check permisons of user identified by Cogntio UI.
def has_permissions(user):
    # TO_DO: Check if user has permisions to do whathener
    return True

```




Finally for logout, we need to splicity go to AWS UI to lgout the User from Cognto:

```py
# Endpoint to logout from our app and from Cognito UI.
@app.route('/logout')
def logout():
    # clean App cookies
    oidc.logout()
    # Logout from Cognito UI
    cognito_logout_uri = "/logout?client_id=" + app.config['COGNITO_UI_CLIENT_ID'] + "&logout_uri=" + app.config['COGNITO_UI_LOGOUT_REDIRECT']
    return redirect(app.config['COGNITO_UI_BASE_URL'] + cognito_logout_uri)

```





