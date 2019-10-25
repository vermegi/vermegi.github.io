---
layout: post
title:  "GitHub Actions First Look"
date:   2019-10-25 14:15:05
categories: azure github functions
background: "/assets/2018-08/superset.PNG"
---

Recently GitHub actions got added to GitHub for adding CI/CD capabilities to both your private and public GitHub repositories. Coming from Azure DevOps myself, I thought this was a nice addition to GitHub to try out. So with this post I'll give you a run-through of how to set this up and some of my findings. 

Enabling GitHub actions for your own GitHub account is as easy as enabling the beta through the [signup page][ghactionssignup] and click 'Sign up for beta'. Once this is done, you will notice all your existing repo's get an addition 'actions' menu item at the top.

![resources]({{ "assets/2019-10/actionsstart.png" | absolute_url }}){: .img-fluid}

To start using these actions, I will create a new project in my GitHub account, you can find my test project [here][vermegiGH]. I added a empty README file and a .gitignore file for node. In this project I will create an Azure function app and try to deploy this to azure through GitHub actions. 

After cloning the project to my local system, I create a Function app project in my directory. I use the func command line tools for this. Instructions on how to install these can be found [here][func]. The below code shows you how to create and start your function project: 

{% highlight cli %}
E:\dev\github\GHActionsDemo\src [master ≡]> mkdir src

E:\dev\github\GHActionsDemo\src [master ≡]> cd src

E:\dev\github\GHActionsDemo\src [master ≡]> func init FunctionProject
Select a worker runtime: node
Select a Language: javascript
Writing package.json
Writing .gitignore
Writing host.json
Writing local.settings.json
Writing E:\dev\github\GHActionsDemo\src\FunctionProject\.vscode\extensions.json

E:\dev\github\GHActionsDemo\src [master ≡ +1 ~0 -0 !]> cd .\FunctionProject\

E:\dev\github\GHActionsDemo\src\FunctionProject [master ≡ +1 ~0 -0 !]> func new
Select a template: HTTP trigger
Function name: [HttpTrigger]
Writing E:\dev\github\GHActionsDemo\src\FunctionProject\HttpTrigger\index.js
Writing E:\dev\github\GHActionsDemo\src\FunctionProject\HttpTrigger\function.json
The function "HttpTrigger" was created successfully from the "HTTP trigger" template.

E:\dev\github\GHActionsDemo\src\FunctionProject [master ≡ +1 ~0 -0 !]> func host start

                  %%%%%%
                 %%%%%%
            @   %%%%%%    @
          @@   %%%%%%      @@
       @@@    %%%%%%%%%%%    @@@
     @@      %%%%%%%%%%        @@
       @@         %%%%       @@
         @@      %%%       @@
           @@    %%      @@
                %%
                %
{% endhighlight %}

The func host start command will point you to a local URL where you can start testing out your function, to check if everything if running correctly.

We want to now deploy this function app to Azure through GitHub actions. My first thought on this was to leverage an ARM template to create my architecture for this in Azure and run this ARM template through GitHub Actions. However, if you look at the current list of [GitHub Actions available for Azure][GHActions], there is not yet an action available for deploying/running ARM templates. 

![resources]({{ "assets/2019-10/GHmarketplace.png" | absolute_url }}){: .img-fluid}

I could have leveraged the 'Azure login' action for this, since this is able to run Azure CLI commands, however, for now I created my function app with local CLI commands: 

{% highlight cli %}
E:\dev\github\GHActionsDemo [master]> az login

E:\dev\github\GHActionsDemo [master ≡ +1 ~0 -0 !]> az group create -n demo-functionapp-GHActions --location westeurope
{
  "id": "/subscriptions/49e23c68-dee0-4c5b-8c21-74acea0a2662/resourceGroups/demo-functionapp-GHActions",
  "location": "westeurope",
  "managedBy": null,
  "name": "demo-functionapp-GHActions",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": null
}

E:\dev\github\GHActionsDemo [master ≡ +1 ~0 -0 !]> az storage account create -n demofunctionappghactions -g demo-functionapp-GHActions -l westeurope --sku Standard_LRS

E:\dev\github\GHActionsDemo [master ≡ +1 ~0 -0 !]> az functionapp create -g demo-functionapp-GHActions --name demofunctionappGHActions --storage-account demofunctionappghactions --runtime node --consumption-plan-location westeurope

{% endhighlight %}

Once you have your function app in place in Azure, you can create a service principal, which we will use in our deployment action in one of the next steps. The service principal needs to be a contributor to your function app: 

{% highlight cli %}
E:\dev\github\GHActionsDemo [master ≡ +1 ~0 -0 !]> az ad sp create-for-rbac -n "gittefuncGHActions" --scopes /subscriptions/xxxx/resourceGroups/demo-functionapp-GHActions/providers/Microsoft.Web/sites/demofunctionappGHActions --sdk-auth
{% endhighlight %}

The Json output of the above command, should be copy-pasted in a new GitHub Secret, with the name AZURE_CREDENTIALS. 

![resources]({{ "assets/2019-10/azure_credentials.png" | absolute_url }}){: .img-fluid}

Once this is in place, in our code repo, we can now add a .github/workflows folder. In this folder you can now create your first GitHub actions workflow in a workflow.yml file. 

{% highlight cli %}
E:\dev\github\GHActionsDemo > mkdir .github/workflows
{% endhighlight %}

Inside this file I copy pasted directly the [sample function action workflow][functionappwf] from the GitHub marketplace. Here I also replaced some of the values as indicated on the sample and I altered some of the paths, since I am using a /src subdirectory in my GitHub project. 

{% highlight cli %}
# Action Requires
# 1. Setup the AZURE_CREDENTIALS secrets in your GitHub Repository
# 2. Replace PLEASE_REPLACE_THIS_WITH_YOUR_FUNCTION_APP_NAME with your Azure function app name
# 3. Add this yaml file to your project's .github/workflows/
# 4. Push your local project to your GitHub Repository

name: Windows_Node_Workflow

on:
  push:
    branches:
    - master

jobs:
  build-and-deploy:
    runs-on: windows-latest
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@master

    # If you want to use publish profile credentials instead of Azure Service Principal
    # Please comment this 'Login via Azure CLI' block
    - name: 'Login via Azure CLI'
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Setup Node 10.x
      uses: actions/setup-node@v1
      with:
        node-version: '10.x'

    - name: 'Run npm'
      shell: pwsh
      run: |
        # If your function app project is not located in your repository's root
        # Please change your directory for npm in pushd
        pushd ./src/FunctionProject
        npm install
        npm run build --if-present
        npm run test --if-present
        popd

    - name: 'Run Azure Functions Action'
      uses: Azure/functions-action@v1
      id: fa
      with:
        app-name: demofunctionappGHActions
        # If your function app project is not located in your repository's root
        # Please consider prefixing the project path in this package parameter
        package: 'src/FunctionProject/.'
        # If you want to use publish profile credentials instead of Azure Service Principal
        # Please uncomment the following line
        # publish-profile: ${{ secrets.SCM_CREDENTIALS }}

    - name: 'use the published functionapp url in upcoming steps'
      run: |
        echo "${{ steps.fa.outputs.app-url }}"

# For more information on GitHub Actions:
#   https://help.github.com/en/categories/automating-your-workflow-with-github-actions
{% endhighlight %}

The above workflow runs on a Windows host, it first checks out my code from GitHub, then it logs into Azure with the secret value I provided, it sets up node, runs a couple of node commands and eventually pushes my code to my Azure function. As a last step in my workflow I have the deployment print out the URL to my function app.

Once I pushed these changes to my repo, I saw my first custom workflow pop up. 

![resources]({{ "assets/2019-10/workflow-run.png" | absolute_url }}){: .img-fluid}

![resources]({{ "assets/2019-10/workflow-run2.png" | absolute_url }}){: .img-fluid}

![resources]({{ "assets/2019-10/workflow-run3.png" | absolute_url }}){: .img-fluid}

Now, unfortunately the URL to my function app, is a bit unusable to test out my function. As you can see in the last picture above, it only has the root URL, but for testing this needs to be appended with the HttpTrigger function name and an API key.

So to test out your function, go to the azure portal, and navigate to your function. Here you have a link for retrieving your function URL. 

![resources]({{ "assets/2019-10/funcurl.png" | absolute_url }}){: .img-fluid}

Add a &name=yourname to the back of this URL to test out your function.

Just to be sure the pipeline was working as expected, I also made a small code change in my index.js file, and pushed that to the repo to see the GitHub actions triggered again. 

{% highlight cli %}
body: "Hello SUPER AWESOME " + (req.query.name || req.body.name)
{% endhighlight %}

{% highlight cli %}
E:\dev\github\GHActionsDemo [master ≡ +0 ~1 -0 !]> git add -A
E:\dev\github\GHActionsDemo [master ≡ +0 ~1 -0 ~]> git commit -m 'altered welcome msg'
[master 6c1d2c9] altered welcome msg
 1 file changed, 1 insertion(+), 1 deletion(-)
E:\dev\github\GHActionsDemo [master ↑1]> git push
Enumerating objects: 11, done.
Counting objects: 100% (11/11), done.
Delta compression using up to 8 threads
Compressing objects: 100% (5/5), done.
Writing objects: 100% (6/6), 504 bytes | 168.00 KiB/s, done.
Total 6 (delta 3), reused 0 (delta 0)
remote: Resolving deltas: 100% (3/3), completed with 3 local objects.
To https://github.com/vermegi/GHActionsDemo.git
   a592ef8..6c1d2c9  master -> master
{% endhighlight %}

And sure, that triggered my workflow and got the change deployed to Azure.

![resources]({{ "assets/2019-10/awesome.png" | absolute_url }}){: .img-fluid}

That was actually pretty easy to set up and get going with it. Also, if you look at the docs of GitHub actions, there are quite a lot of parallel workflows you can run per GitHub repository ([up to 20][usage]), that is actually more than what you get by default by Azure Pipelines. 

On the other hand, the amount of pre-built actions is still limited, but I have heard (through the grapevine) the community is very involved in adding new actions. 

Also interface-wise, getting an overview of all your runs, ie. navigating back from the detail of a run, to the list of runs, isn't possible in the current UI. You need to navigate back to your full list of workflows. But, hey, if that's the only flaw, I am very happy to live with that one.

Already nice work from the team! Happy to see this evolve in more!

Also, as an addendum. Once you add GitHub actions to your Github account, you get different proposals of workflows that would make sense to your repository. I clicked open the actions menu for this blog (which lives on GitHub pages and is compiled through Jekyll) and first recomendation for an action workflow was Jekyll docker build. Nice!!!


[ghactionssignup]: https://github.com/features/actions
[func]: https://github.com/Azure/azure-functions-core-tools
[vermegiGH]: https://github.com/vermegi/GHActionsDemo
[GHActions]: https://github.com/marketplace?type=actions
[functionappwf]: https://raw.githubusercontent.com/Azure/actions-workflow-samples/master/windows-node.js-functionapp-on-azure.yml
[usage]: https://help.github.com/en/github/automating-your-workflow-with-github-actions/about-github-actions#usage-limits