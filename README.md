# flutter-azure-example

Heavily inspired by a Medium article written by Himanshu Sharma, [deploy-flutter-web-app-to-azure-app-service-with-node-js](https://medium.com/flutter-community/deploy-flutter-web-app-to-azure-app-service-with-node-js-b0781fc6def2)

---

### An overview of steps:

Setup takes a few minutes, but once you're done you should be set up for future deployments.

1. **Setup a Git(hub) repository:**

   Your code repository, which when setup will also handle any deployments to Azure.

   
2. **Create an Azure Web App:**

   The web app that will host your Flutter web app, using Node as the "server".


3. **Flutter project configuration**

   Create / use existing Flutter project & prepare it for deployment.


4. **Node.js configuration**

   Node.js server setup & configuration serving Flutter web app.


5. **CI/CD Pipeline**

   Configure Github repository to trigger a deployment to Azure when code is pushed.


6. **Deploy to Azure**

   Commit & push your code to your repository, which will trigger a deployment to Azure.

---

#### 1. Setup a Git(hub) repository
This step is only necessary if you don't already have a repository setup. If you do, skip to step 2.

1. Create a new repository on Github (or any other Git provider)

#### 2. Create Azure Code Web app (or Web Resource) 
Expect to find yourself on this page: [https://portal.azure.com/?quickstart=true#create/Microsoft.WebSite](https://portal.azure.com/?quickstart=true#create/Microsoft.WebSite)
1. Fill out information on **Basics** page
2. In `Deployment` section, link your Github repository to setup CI/CD pipeline (this includes Github secrets, which is easier setup by Azure themselves)
3. Click **Review + Create**

**Important (if Github account is linked)**
If you review your Github project commits, you will see a new commit with a message similar to `Add or update the Azure App Service build and deployment workflow config`
This is the CI/CD pipeline that Azure has setup for you. It will automatically deploy your code to Azure when you push to your repository.

Clone your repository or pull latest changes into your Flutter project directory (or when we create a new Flutter project in the next step, clone your repository into that directory)

After doing so, you should see a new directory called `.github` with a `workflows` directory inside. We will configure this in step 5.

### 3. Flutter setup
1. Create a new Flutter project (or use an existing one)
   - Create a new Flutter project with `flutter create <project_name> --platforms=web`
2. Ensure you have a `web` folder in your project (if not, enable web support with `flutter config --enable-web`)
3. Test your app locally with `flutter run -d chrome` (just to be sure it works)

### 4. Create a Node.js server
We will use express to serve our Flutter web app, however you can use any server you want. For this example, we will also use express generator to get us setup.

1. Install express generator with `npm install -g express-generator`
2. Create a new express project with (with bare minimum):
   - `express --no-view` (if you don't want to use the default view engine)
   - `rm -r public routes` (remove the public and routes folders)
   
3. In `app.js`, we need to make a few small changes:
   1. Remove references to existing routes:
   ```js
   var indexRouter = require('./routes/index');
   var usersRouter = require('./routes/users');
   ...
   app.use('/', indexRouter);
   app.use('/users', usersRouter);
   ```
   
   2. Reassign `public` directory to Flutter's web build directory:
   ```js
   app.use(express.static(path.join(__dirname, 'build/web')));
   ```
   _Note: `build/<platform>` directories are created when running `flutter build <platform>`, the `web` directory is created when running `flutter build web`_

4. Install dependencies with `npm install`, then
5. Test locally with `npm start` (just to be sure it works), then navigate to `http://localhost:3000` in your browser

### 5. CI/CD Pipeline

Now that we have a Flutter project with a `build/web` directory, and a Node.js server that serves the contents of that directory, we can setup our CI/CD pipeline.

1. In your Github repository, navigate to `.github/workflows/<azure-workflow-from-step-1>.yml`
2. Since this workflow file is aimed at Node applications, we need to ensure that it can build flutter apps too. To do so, we need to setup the Flutter environment, enable web configuration and build the web project 

Inside of the workflow file, there are 2 jobs: `build` and `deploy`. We will make changes to `build` only.

```yml
# Docs for the Azure Web Apps Deploy action: https://github.com/Azure/webapps-deploy
# More GitHub Actions for Azure: https://github.com/Azure/actions

name: Build and deploy Node.js app to Azure Web App - twilio-voice-web

on:
  push:
    branches:
      - main # your github branch name, can be production, master, etc.
  workflow_dispatch: # allows you to manually run workflow from github 

jobs:
  build: # we will make changes to this one only
    runs-on: ubuntu-latest
    ...

  deploy:
    runs-on: ubuntu-latest
    ...
```

lets look deeper into the `build` job:

```yml
build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
        
      # Here we need to add the Flutter environment setup steps, since Node will automatically build the project, our Flutter project is a 'dependency' and needs to be built first
      # <insert flutter setup steps here>

      - name: Set up Node.js version
        uses: actions/setup-node@v1
        with:
          node-version: '18.x'

      - name: npm install, build, and test
        run: |
          npm install
          npm run build --if-present
          npm run test --if-present
      - name: Upload artifact for deployment job
        uses: actions/upload-artifact@v2
        with:
          name: node-app
          path: .
```

In the above workflow, lets add the following steps replacing the `# <insert flutter setup steps here>` text`:
    
```yml
- uses: actions/checkout@v2 # DO NOT COPY THIS, THIS IS A REFERENCE POINT ONLY

# --- ADD HERE START --- 
- name: Setup Flutter build environment # Custom name of workflow step we'll see in the logs
  uses: subosito/flutter-action@v2
  with:
    flutter-version: '3.7.11' # The flutter version we use (in the 'stable' branch), change as needed
    channel: 'stable'
    cache: true # Cache the flutter SDK between builds, speeds up build time
- run: flutter --version # Print the flutter version to the logs
- run: flutter pub get # Get the dependencies, needed to build the project
- run: flutter config --enable-web # Enable web support for the project
- run: flutter build web --release --web-renderer=html # Build the web project in release mode. We're using the html renderer, but you can use canvaskit if you want (html has better performance on mobile)
### --- ADD HERE END --- 

- name: Set up Node.js version # DO NOT COPY THIS, THIS IS A REFERENCE POINT ONLY
```

Now we've created our Flutter project, got a HTTP node server setup, configured for `build/web` directory, and setup our CI/CD pipeline to build our Flutter project and deploy it to Azure.
   
### 6. Deploy to Azure

1. Push your changes to Github
2. Browse to your Github repository, and navigate to the `Actions` tab and observe a new workflow running. 
_Once completed, you should see a green checkmark next to it and your site should be live._

Navigate to your Azure Web App, and click on the URL to view your site (should also be provided at the end of the deploy workflow logs)

e.g. https://your-project-id.azurewebsites.net/ 