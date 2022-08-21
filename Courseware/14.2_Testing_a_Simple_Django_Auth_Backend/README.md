# Testing a Simple Django Auth Backend

In the latest repository, a new file `connected-devices/Web/lampisite/lampi/mqtt_auth_views.py` has been added that provides three new Django view functions for the Mosquitto auth plugin backend:

* `auth` - authentication check
* `superuser` - superuser check (if successful, all ACL checks bypassed)
* `acl` - ACL check

`connected-devices/Web/lampisite/lampi/urls.py` has been updated with URL paths to support these views.

The code looks like:

```python
from django.http import HttpResponse, HttpResponseForbidden
from django.views.decorators.csrf import csrf_exempt
from django.contrib.auth import authenticate
from django.contrib.sessions.models import Session
from django.contrib.auth.models import User
from paho.mqtt.client import topic_matches_sub


ACL_READ = "1"  
ACL_WRITE = "2"  
ACL_SUBSCRIBE = "4"


def unpack_POST_data(req):
    post_data = req.POST

    username = post_data.get("username", "")
    password = post_data.get("password", "")
    topic = post_data.get("topic", "")
    acc = post_data.get("acc", "")
    clientid = post_data.get("clientid", "")

    print("  UNPACKED POST: U: '{}' P: '{}'"
          " T: '{}' A: '{}' C: '{}'".format(username,
                                            password,
                                            topic,
                                            acc,
                                            clientid))

    return username, password, topic, acc, clientid


def get_user_from_session_key(session_key):
    try:
        session = Session.objects.get(session_key=session_key)
    except Session.DoesNotExist:
        return None
    uid = session.get_decoded().get('_auth_user_id')
    try:
        return User.objects.get(pk=uid)
    except User.DoesNotExist:
        return None


@csrf_exempt
def auth(req):
    if req.method == 'POST':
        if req.META['REMOTE_ADDR'] == '127.0.0.1':
            username, password, topic, acc, clientid = unpack_POST_data(req)
            # need to handle
            #   django users - with username and password (mqtt-daemon)
            #   django users - with websockets
            #      username: device_id
            #      password: django session_key
            #   LAMPI devices - brokers authenticated with Certificates
            #      mosquitto handles SSL/TLS auth directly, so this
            #      auth function never is invoked
            #
            return HttpResponse(None, content_type='application/json')

    return HttpResponseForbidden(None, content_type='application/json')


@csrf_exempt
def superuser(req):
    # do not allow any superusers
    return HttpResponseForbidden(None, content_type='application/json')


@csrf_exempt
def acl(req):
    if req.method == 'POST':
        if req.META['REMOTE_ADDR'] == '127.0.0.1':
            username, password, topic, acc, clientid = unpack_POST_data(req)
            # need to handle
            #   django users
            #   django superusers
            #   websockets users
            #   LAMPI devices - broker bridges
            return HttpResponse(None, content_type='application/json')

    return HttpResponseForbidden(None, content_type='application/json')
```

### Code Explanation

There are five functions:

1. `unpack_POST_data(request)` - a helper function to unpack the 5 parameters sent in each request from the auth plugin; it will print out the parameters to the console for easy debugging (note: some will be blank for certain operations)
1. `get_user_from_session_key` - a helper function that might be helpful for the assignment
2. `auth(request)` - the authentication view - it should return a `HttpResponse` on approve (200) or a `HttpResponseForbidden` (403) on deny
3. `superuser(request)` - the supervisor check view - it returns `HttpResponseForbidden` (403) on deny on all requests - we are not supporting superusers for our system
4. `acl(request)` the Access Control check view - it should return a `HttpResponse` on approve (200) or a `HttpResponseForbidden` (403) on deny

We only want this service to be accessible by the Mosquitto auth plugin running on localhost, so we deny access (`HttpResponseForbidden`) if the client is not on localhost (127.0.0.1) or if the request is not a POST.


**NOTE:** Disable 'memcached' during development - since we are caching the entire site, it will cache our authentication and ACL responses, too, leading to confusing behavior as you change the code but the system response does not change since it is serving up the cached response

```bash
$cloud sudo service memcached stop
```

If Django cannot reach 'memcached' at runtime it will silently fall back to non-cache behavior.

Hint: the other imports at the top of the file _might_ be useful for the assignment.

## Setup commandline on the Cloud for testing the Auth Plugin

For testing, let's open up 3 SSH sessions:

1. EC2 for Django mqtt-auth
1. EC2 for mosquitto
1. LAMPI for mosquitto


On your EC2 Django SSH:

```bash
$cloud cd ~/connected-devices/Web/lampisite
$cloud ./manage.py runserver 0.0.0.0:8080
```

On your EC2 mosquitto SSH:

```bash
$cloud sudo service mosquitto stop
$cloud sudo mosquitto -c /etc/mosquitto/mosquitto.conf -v
```

On your lamp SSH:

```bash
$lamp sudo service mosquitto stop
$lamp sudo mosquitto -c /etc/mosquitto/mosquitto.conf -v
```

You should see many messages showing up in the EC2 SSH sessions.  For example, the EC2 Django session should have output something like:

```
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
April 18, 2022 - 01:05:15
Django version 4.0.2, using settings 'lampisite.settings'
Starting development server at http://0.0.0.0:8080/
Quit the server with CONTROL-C.
Forbidden: /lampi/superuser
[18/Apr/2022 01:05:20] "POST /lampi/superuser HTTP/1.1" 403 4
U: 'b827eb8f3d71_broker' P: '' T: '$SYS/broker/connection/b827eb8f3d71_broker/state' A: '2' C: 'b827eb8f3d71_broker'
[18/Apr/2022 01:05:20] "POST /lampi/acl HTTP/1.1" 200 4
Forbidden: /lampi/superuser
[18/Apr/2022 01:05:20] "POST /lampi/superuser HTTP/1.1" 403 4
U: 'mqtt-daemon' P: '' T: '$SYS/broker/connection/b827eb8f3d71_broker/state' A: '1' C: 'mqtt-daemon'
[18/Apr/2022 01:05:20] "POST /lampi/acl HTTP/1.1" 200 4
Forbidden: /lampi/superuser
[18/Apr/2022 01:05:20] "POST /lampi/superuser HTTP/1.1" 403 4
U: 'b827eb8f3d71_broker' P: '' T: 'devices/b827eb8f3d71/lamp/set_config' A: '4' C: 'b827eb8f3d71_broker'
[18/Apr/2022 01:05:20] "POST /lampi/acl HTTP/1.1" 200 4
Forbidden: /lampi/superuser
[18/Apr/2022 01:05:20] "POST /lampi/superuser HTTP/1.1" 403 4
U: 'b827eb8f3d71_broker' P: '' T: 'devices/b827eb8f3d71/lamp/associated' A: '4' C: 'b827eb8f3d71_broker'
[18/Apr/2022 01:05:20] "POST /lampi/acl HTTP/1.1" 200 4
Forbidden: /lampi/superuser
[18/Apr/2022 01:05:20] "POST /lampi/superuser HTTP/1.1" 403 4
U: 'b827eb8f3d71_broker' P: '' T: 'devices/b827eb8f3d71/lamp/associated' A: '1' C: 'b827eb8f3d71_broker'
[18/Apr/2022 01:05:20] "POST /lampi/acl HTTP/1.1" 200 4
```

where the lines beginning `U` are the `print()` output from `unpack_POST_data`, and the other lines are the subsequent HTTP transaction summary with result code (200s in this case).

and the Mosquitto output should look like:

```
DEBU[2022-04-18T01:05:20Z] Superuser check with backend HTTP
INFO[2022-04-18T01:05:20Z] error code: 403
DEBU[2022-04-18T01:05:20Z] Acl check with backend HTTP
DEBU[2022-04-18T01:05:20Z] http request approved for b827eb8f3d71_broker
DEBU[2022-04-18T01:05:20Z] user b827eb8f3d71_broker acl authenticated with backend HTTP
DEBU[2022-04-18T01:05:20Z] Acl is true for user b827eb8f3d71_broker
DEBU[2022-04-18T01:05:20Z] Superuser check with backend HTTP
INFO[2022-04-18T01:05:20Z] error code: 403
DEBU[2022-04-18T01:05:20Z] Acl check with backend HTTP
DEBU[2022-04-18T01:05:20Z] http request approved for mqtt-daemon
DEBU[2022-04-18T01:05:20Z] user mqtt-daemon acl authenticated with backend HTTP
DEBU[2022-04-18T01:05:20Z] Acl is true for user mqtt-daemon
DEBU[2022-04-18T01:05:20Z] Superuser check with backend HTTP
INFO[2022-04-18T01:05:20Z] error code: 403
DEBU[2022-04-18T01:05:20Z] Acl check with backend HTTP
DEBU[2022-04-18T01:05:20Z] http request approved for b827eb8f3d71_broker
DEBU[2022-04-18T01:05:20Z] user b827eb8f3d71_broker acl authenticated with backend HTTP
DEBU[2022-04-18T01:05:20Z] Acl is true for user b827eb8f3d71_broker
DEBU[2022-04-18T01:05:20Z] Superuser check with backend HTTP
INFO[2022-04-18T01:05:20Z] error code: 403
DEBU[2022-04-18T01:05:20Z] Acl check with backend HTTP
DEBU[2022-04-18T01:05:20Z] http request approved for b827eb8f3d71_broker
DEBU[2022-04-18T01:05:20Z] user b827eb8f3d71_broker acl authenticated with backend HTTP
DEBU[2022-04-18T01:05:20Z] Acl is true for user b827eb8f3d71_broker
DEBU[2022-04-18T01:05:20Z] Superuser check with backend HTTP
INFO[2022-04-18T01:05:20Z] error code: 403
DEBU[2022-04-18T01:05:20Z] Acl check with backend HTTP
DEBU[2022-04-18T01:05:20Z] http request approved for b827eb8f3d71_broker
DEBU[2022-04-18T01:05:20Z] user b827eb8f3d71_broker acl authenticated with backend HTTP
DEBU[2022-04-18T01:05:20Z] Acl is true for user b827eb8f3d71_broker
```

Which is a detailed summary of the superuser and ACL checks (note the `403` in both outputs, for instance - that is the superuser check failing)


The `/auth` view is only accessed when a client connecting provides a username and password (recall that we are now disallowing all anonymous connections).  The LAMPI bridge connection bypasses the authentication (username/password) because we are using SSL/TLS Certificates for mutual client/server authentication.

Next up: go to [14.3 Using Django for Authentication & Authorization](../14.3_Django_Authentication_and_Authorization/README.md)

&copy; 2015-2022 LeanDog, Inc. and Nick Barendt
