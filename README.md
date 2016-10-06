# Python Minichat

Minimal chat example using python on App Engine.

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

- url: /.*
  script: server.app

libraries:
- name: webapp2
  version: latest
- name: jinja2
  version: latest
```
