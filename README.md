# Python Minichat

Minimal chat example using python on App Engine.

Deployed at <https://python-minichat-145600.appspot.com/>

### 1. Create new project on App Engine

Go to <https://console.cloud.google.com> and click on CREATE PROJECT

![create project button](img/create-project-button.png)

Choose a project name for your project:

![new project dialog](img/new-project-dialog.png)

Install the Google Cloud Tools from here <https://cloud.google.com/sdk/docs/>

Once installed run

    gcloud init
    
Login and select the project you just created.

### 2. Create Python Backend

In your project directory create an `app.yaml` file with the following contents:

```yaml
runtime: python27
api_version: 1
threadsafe: true

handlers:
- url: /static
  static_dir: static
- url: /.*
  script: server.app
- url: /_ah/channel/disconnected/
  script: server.app

inbound_services:
- channel_presence

libraries:
- name: webapp2
  version: latest
- name: jinja2
  version: latest
```

Create a `server.py` file. This will be where the code for your backend python
application lives. The first it needs to do is import the libraries it will
need.

```python
import os
import jinja2
import webapp2

from google.appengine.api import channel
from google.appengine.api import users
from google.appengine.ext import ndb
```

Next it needs to set up the jinja libaries template system. This will help
serve and render serve the `index.html` file later on.

```python
JINJA_ENVIRONMENT = jinja2.Environment(
    loader=jinja2.FileSystemLoader(os.path.dirname(__file__)),
    extensions=['jinja2.ext.autoescape'],
    autoescape=True)
```

Next we need to add our main application:

```python
class MainHandler(webapp2.RequestHandler):
  def get(self):
    # Check if user is logged in
    user = users.get_current_user()
    if user:
      # Create unique chat channel for user and save to active list
      token = channel.create_channel(user.user_id())
      # Add user to active users
      ActiveUser(userid=user.user_id(), id=user.user_id()).put()
      # Render index.html for user
      template = JINJA_ENVIRONMENT.get_template('index.html')
      self.response.write(template.render({'token': token}))
    else:
      self.redirect(users.create_login_url(self.request.uri))

app = webapp2.WSGIApplication([
    ('/', MainHandler),
], debug=True)
```

Now you might notice an `ActiveUser` reference that wasn't defined or imported.
Let's define it! Before `MainHandler` add:


```python
class ActiveUser(ndb.Model):
  userid = ndb.StringProperty()
```

This will help keep track of users currently chatting on our site by storing
them in the App Engine datastore.

Now our app needs a handler for the site to actually send chats to. When a user
send a chat to the backend this handler will send out that message to each
currently active user.

```python
class ChatHandler(webapp2.RequestHandler):
  def post(self):
    user = users.get_current_user()
    message = self.request.body
    if message:
      formatted_message = '%s: %s' % (user.nickname(), message)
      for activeuser in ActiveUser.query():
        channel.send_message(activeuser.userid, formatted_message)
```

And register the url for this handler in the `app` creation at the bottom of the
file.

```python
app = webapp2.WSGIApplication([
    ('/', MainHandler),
    ('/chat', ChatHandler),
], debug=True)
```

Almost done with the backend, just one more handler. Note that the app adds
`ActiveUsers` when they load the site but never removes them. So our app isn't
trying to send chat messages to long gone users we need to add a disconnect
handler.

```python
class DisconnectHandler(webapp2.RequestHandler):
  def post(self):
    userid = self.request.get('from')
    if userid:
      ndb.Key(ActiveUser, userid).delete()
```

Then in the `app` creation add the disconnect url that App Engine uses to notify
our app that a user disconnected from our channel.

```python
app = webapp2.WSGIApplication([
    ('/', MainHandler),
    ('/chat', ChatHandler),
    ('/_ah/channel/disconnected/', DisconnectHandler),
], debug=True)
```




