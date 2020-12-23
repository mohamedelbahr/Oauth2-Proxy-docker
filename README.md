# Oauth2-Proxy-docker

This docker-compose environment is used to proxy requests for certain backend web application over Gmail authentication of a certain domain **"ALLOWED_DOMAIN"** variable

## Running the app ##

1. Clone the repository

2. Re-name `.env.example` file to `.env` and replace included variables as per described comments

3. Run `docker-compose up -d`

## Configurations

### A) Google auth provider configurations


1. Create a new project: https://console.developers.google.com/project

2. Choose the new project from the top right project dropdown (only if another project is selected)

3. In the project Dashboard center pane, choose "API Manager"

4. In the left Nav pane, choose "Credentials"

5. In the center pane, choose "OAuth consent screen" tab. Fill in "Product name shown to users" and hit save.

6. In the center pane, choose "Credentials" tab.

      a. Open the "New credentials" drop down
     
      b. Choose "OAuth client ID"
      
      c. Choose "Web application"
      
      d. Application name is freeform, choose something appropriate
      
      e. Authorized JavaScript origins is your domain ex: https://internal.yourcompany.com
      
      f. Authorized redirect URIs is the location of oauth2/callback ex: https://internal.yourcompany.com/oauth2/callback
      
      g. Choose "Create"

7. Take note of the Client ID and Client Secret


** For more info visit this [link](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider/#google-auth-provider)


### B) Main Proxy Configurations

- Assuming you are using **nginx** as a revers proxy to web backends, Below is an example of virtual host file syntax that proxy authentication requests to our oauth2 proxy before it reaches the desired backend: *(please consider comments)*

```bash
# The backend app is running on the same server on port xyzw 
    upstream backend_1 {
        server 127.0.0.1:xyzw;
    }
    # Configuration for the server
    server {
        server_name example.webserver.com;

# Assuming the oauth2 proxy is running on external port 4180 
# (The same as internal one, LISTEN_ADDRESS=0.0.0.0:4180)
        location /oauth2/ {
            proxy_pass http://127.0.0.1:4180;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Scheme $scheme;
            proxy_set_header X-Auth-Request-Redirect $request_uri;
        }

        location / {

            auth_request /oauth2/auth;
            error_page 401 = /oauth2/sign_in;
            # pass information via X-User and X-Email headers to backend
            # requires running with --set-xauthrequest flag auth_request_set $user $upstream_http_x_auth_request_user; 
            auth_request_set $email $upstream_http_x_auth_request_email;
            auth_request_set $user   $upstream_http_x_auth_request_user;

            proxy_set_header X-User $user;
            proxy_set_header X-Email $email;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_pass_header Server;
            proxy_connect_timeout 3s;
            proxy_read_timeout 10s;

            # if you enabled --cookie-refresh, this is needed for it to work with auth_request
            auth_request_set $auth_cookie $upstream_http_set_cookie; add_header Set-Cookie $auth_cookie;
            proxy_pass http://backend_1;

        }

    listen 443 ssl; 
    ssl_certificate /some/path/to/certificate/file;
    ssl_certificate_key /some/path/to/certificate/key;
}

```
