**How to do it:**

The `query-test` GitHub App has an endpoint to a server that is listening to POST requests. That server has Python installed with the `jwt` library. It uses `jwt` to convert the private key associated with the GitHub App into a JWT token, which has a short expiration date (10 minutes). That token can be made like this:

```python
import jwt
import time
import requests

APP_ID = ...
PRIVATE_KEY_PATH = ...

def generate_jwt(app_id, private_key_path):
    # Current time and expiration time
    now = int(time.time())
    expiration_time = now + (10 * 60)
    payload = {
        'iat': now,
        'exp': expiration_time,
        'iss': app_id
    }
    # Read the private key
    with open(private_key_path, 'rb') as key_file:
        private_key = jwt.jwk_from_pem(key_file.read())
    instance = jwt.JWT()
    token = instance.encode(payload, private_key, alg='RS256')
    return token

def get_installation_token(jwt, installation_id):
    headers = {
        'Authorization': f'Bearer {jwt}',
        'Accept': 'application/vnd.github+json'
    }
    url = f'https://api.github.com/app/installations/{installation_id}/access_tokens'
    response = requests.post(url, headers=headers)
    response_data = response.json()
    return response_data.get('token')

jwt_token = generate_jwt(APP_ID, PRIVATE_KEY_PATH)
installation_id = ...
installation_token = get_installation_token(jwt_token, installation_id)
print(installation_token)
```

where the `APP_ID`, `PRIVATE_KEY_PATH`, and `installation_id` all come from the GitHub App (and where I put its private key in the filesystem).

A process is always listening to the URL and ready to respond to any Discussion comment creation triggers with a Discussion comment creation request, like this:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json
import threading
import requests
import pprint

class RequestHandler(BaseHTTPRequestHandler):
    requests = []
    def do_POST(self):
        self.send_response(200)
        self.end_headers()
        content_length = int(self.headers["Content-Length"])
        post_data = self.rfile.read(content_length).decode("utf-8")
        RequestHandler.requests.append(post_data)
        payload = json.loads(post_data)
        pprint.pprint(payload)
        if payload["comment"]["parent_id"] is None:
            replyto = "replyToId: \"" + payload["comment"]["node_id"] + "\","
        else:
            replyto = ""
        if payload.get("action") == "created" and "comment" in payload:
            headers = {
                "Authorization": "token JWT_TOKEN",
                "Accept": "application/vnd.github+json"
            }
            api_url = "https://api.github.com/graphql"
            data = ('''mutation AddDiscussionComment {
                  addDiscussionComment(input: {
                    discussionId: "%s",
                    %s
                    body: "(quote %s)"
                  }) {
                    comment {
                      id
                    }
                  }
                }''' % (payload["discussion"]["node_id"], replyto, payload["comment"]["body"]))
            print(data)
            response = requests.post(api_url, json={"query": data}, headers=headers)
            print(f"{response.status_code = } {response.text = }")
        self.send_response(200)
        self.end_headers()

def run():
    server = HTTPServer(("", 8000), RequestHandler)
    server.serve_forever()

server_thread = threading.Thread(target=run)
server_thread.start()
```

where `JWT_TOKEN` is the JWT token returned by the first process. When anyone creates a Discussion comment, this `do_POST` runs, and the `requests.post` creates another Discussion comment. Although it's possible to reply to a top-level comment by its `node_id`, there doesn't seem to be any way to respond to a reply. I'd want to provide the GitHub Global Node ID of the parent, but I only have a numeric (database) ID for the parent, and there's no way to convert between them. GraphQL, which is the only way to make Discussion comments (without a "team slug"?!?), requires node ids, and while the trigger event gives me a node id for a top-level comment, it does not give me a node id for the parent of a reply (i.e. the top-level comment that this should be sent to). Nesting a reply in a reply is not allowed.

**Update:**

This could be a triggered GitHub Action, rather than an independent service (e.g. on AWS) that we have to maintain.

https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#discussion_comment

The webhook URL can then be set to "inactive."
