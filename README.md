# Applanga Github Workflow Integration Example
 
The example [repository](https://github.com/applanga/github-workflow-example) showcases a full cycle [github workflow setup](https://github.com/applanga/setup-applanga-cli) integration with the [Applanga Localization Platform](https://www.applanga.com).

The benefit of using github workflows is that you can automate your localization process without the need to share any repository credentials with your localization provider.

To use github workflows on your repository you need to create a folder call .github/workflows/ and place the workflow configuration .yml files in there. 

For a more detailed introduction to github workflows please see the [Github Documentation](https://help.github.com/en/actions/configuring-and-managing-workflows/configuring-a-workflow)

---
# Workflow Configurations

The [repository](https://github.com/applanga/github-workflow-example) contains 3 sample workflows. Running and past workflow actions can be tracked under the [**Actions** Tab](https://github.com/applanga/github-workflow-example/actions).

[.github/workflows/applanga-push.yml](https://github.com/applanga/github-workflow-example/blob/master/.github/workflows/applanga-push.yml) will push any `.json` source files under the directory `react_json_sample/en/` to Applanga whenever they are changed in the repository. Depending on your folder structure and file format you need to modify the `paths` for your repository workflow config.

```yaml
name: "Push Source Files to Applanga"
on:
 push:
   branches:
     - master
   paths:
     - react_json_sample/en/*.json
   
jobs:
  push-sources-for-translation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          path: 'checkout'
      - uses: applanga/setup-applanga-cli@v1.0.1
        with:
          version: 1.0.48
      - name: Push Sources to Applanga
        env: 
         APPLANGA_ACCESS_TOKEN: ${{ secrets.APPLANGA_ACCESS_TOKEN }}
        run: applanga push --force
        working-directory: checkout
```

Note that for the above workflow file the `Applanga CLI` will only get triggered when there is a push to the `master` branch as specified in the config. It is also possible to specify a different branch or add more branches as needed.

[.github/workflows/applanga-pull.yml](https://github.com/applanga/github-workflow-example/blob/master/.github/workflows/applanga-pull.yml) pulls new translations available from Applanga and then creates a pull request in the repo with the newly added languages or updated translation files. For this to work a **Webhook Endpoint** has to be configured for the project on Applanga dashboard to trigger the workflow. See [Configure Webhook Endpoint](#configure-webhook-endpoint) below for how to configure 
a webhook endpoint.

```yaml
name: "Pull Target Files from Applanga"
on:
  repository_dispatch:
    types: [applanga-pull]
jobs:
  pull-translation-in:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          ref: ${{ github.head_ref }}
          path: 'checkout'
      - uses: applanga/setup-applanga-cli@v1.0.1
        with:
          version: 1.0.48
      - name: Pull translations from Applanga
        env: 
         APPLANGA_ACCESS_TOKEN: ${{ secrets.APPLANGA_ACCESS_TOKEN }}
        run: applanga pull
        working-directory: checkout
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v3
        with:
          branch: newTranslations
          author: github-actions[bot] <41898282+github-actions[bot]@users.noreply.github.com>
          commit-message: Updated translations
          title: Updated translations 
          body: Pulled in new translations from Applanga portal
          path: 'checkout'
```

****Caveat** The preceding workflow only gets run on the default branch, this is because the `repository_dispatch` event does not get triggered for a non default branch. More information on this can be found in the following  [github issues page](https://github.community/t/how-to-trigger-repository-dispatch-event-for-non-default-branch/14470).
However there is a way to achieve this using the `workflow_dispatch` event described [here](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch). For this to work though the configuration file has to first be created from the default branch. In other words start first by adding the workflow file to the `master` branch of the repository, which allows github to pick up the workflow. Next you can create a branch of the master branch and update the workflow file as needed. See the following sample workflow config below.

[.github/workflows/applanga-pull-branch.yml](https://github.com/applanga/github-workflow-example/blob/master/.github/workflows/applanga-pull-branch.yml) To run this workflow an api request should be made to Github specifying either the file name(`applanga-pull-branch.yml`) or workflow id. Please refer to [Configure Webhook Endpoint to trigger workflow in a branch](#configure-webhook-endpoint-to-trigger-workflow-in-a-branch) for a detailed example of how to do this.

```yaml
name: "Pull Target Files from Applanga on specific branch"
on: workflow_dispatch
jobs:
  pull-translation-in:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          ref: ${{ github.head_ref }}
          path: 'checkout'
      - uses: applanga/setup-applanga-cli@v1.0.1
        with:
          version: 1.0.48
      - name: Pull translations from Applanga
        env: 
         APPLANGA_ACCESS_TOKEN: ${{ secrets.APPLANGA_ACCESS_TOKEN }}
        run: applanga pull
        working-directory: checkout
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v3
        with:
          branch: newTranslations
          author: github-actions[bot] <41898282+github-actions[bot]@users.noreply.github.com>
          commit-message: Updated translations
          title: Updated translations 
          body: Pulled in new translations from Applanga portal
          path: 'checkout'
```

---
## Configure Webhook Endpoint
To the trigger the `applanga-pull` workflow a webhook endpoint has to be configured in the projects settings page on applanga. 

The webhook is triggered at least 15 minutes from when there is no translation change. This means whenever translation is added or edited a webhook request is scheduled to be sent to all configured endpoints for the project 15 minutes later. The scheduled request will be sent as planned unless there is a new change to translation before the scheduled time, in which case it is rescheduled to be sent 15 minutes later.

Here are the steps to setup the **Webhook Endpoint**
* Login to Applanga dashboard and navigate to the project. Then click on **Project Settings**

![](https://www.applanga.com/assets/images/docu/groups_editapp.png)

* In the settings page scroll down to the section **WEB HOOKS** and click the **Add endpoint** button, this will show a modal where the endpoint values can be entered
![](https://www.applanga.com/assets/images/docu/webhook_settings.png)

The values should be configured as follows
- **URL** Insert the following endpoint `https://api.github.com/repos/<owner>/<repo>/dispatches`. Notice the placeholder values `<owner>`(github username) and `<repo>`(repository name) should be replaced with the correct value for your usecase.
- **POST** is selected as http method
- **Headers** Notice in the screenshot below that 2 headers were set, they are `Authorization` and `Accept`. The `Authorization` header value is a valid github access token with repository permissions combined like so `token <Access token>`. For example if you had an access token `ghp_zWAdtUqmtFTY7qkqJS1wmuEx6t4X0SpIPv` then the Authorization header would be set like so `token ghp_zWAdtUqmtFTY7qkqJS1wmuEx6t4X0SpIPv`. Please refer to the following github documentation [here](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) on how to generate an access token.
- **JSON** should be selected in the body tab
- `{"event_type":"applanga-pull"}` is entered in the **Request body**

![](https://www.applanga.com/assets/images/docu/webhook_endpoint_header.png)

![](https://www.applanga.com/assets/images/docu/webhook_endpoint_body.png)

You can test the configured endpoint to make sure everything works well by clicking the **Test endpoint** button top right.
If the config is done correctly the test result should look like the screeshot below

![](https://www.applanga.com/assets/images/docu/webhook_endpoint_test.png)

Click ***Save endpoint*** and you're done!.

You can find more information on creating a github repository dispatch event via REST API [here](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event)

---

## Configure Webhook Endpoint to trigger workflow in a branch
To configure a webhook endpoint to trigger a workflow in a specific branch follow these steps

* In the settings page scroll down to the section **WEB HOOKS** and click the **Add endpoint** button, this will show a modal where the endpoint values can be entered
* Set the http method to *POST* and enter the endpoint url as follows `https://api.github.com/repos/<owner>/<repo>/actions/workflows/{workflow_id}`. The following values `<owner>`, `<repo>` and `{workflow_id}` should be replaced with the correct values. The `workflow_id` can be the id of the workflow you want to trigger or the name of the workflow instead.
To get the list of workflows for a repository a GET request should be made to the following endpoint
`https://api.github.com/repos/OWNER/REPO/actions/workflows`, note that `OWNER` and `REPO` should be replaced with the correct values. More information on this endpoint can be found [here](https://docs.github.com/en/rest/actions/workflows#list-repository-workflows). Find a sample curl request and response below

```curl
curl --location --request GET 'https://api.github.com/repos/oaks-view/github-workflow-example/actions/workflows' \
--header 'Authorization: token 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' \
--header 'accept: application/vnd.github.v3+json'
```

```json
{
    "total_count": 3,
    "workflows": [
        {
            "id": 24995310,
            "node_id": "W_kwDOGBq-Vc4BfWXu",
            "name": "Pull Target Files from Applanga on specific branch",
            "path": ".github/workflows/applanga-pull-branch.yml",
            "state": "active",
            "created_at": "2022-04-27T16:41:47.000+02:00",
            "updated_at": "2022-04-28T14:14:22.000+02:00",
            "url": "https://api.github.com/repos/oaks-view/github-workflow-example/actions/workflows/24995310",
            "html_url": "https://github.com/oaks-view/github-workflow-example/blob/master/.github/workflows/applanga-pull-branch.yml",
            "badge_url": "https://github.com/oaks-view/github-workflow-example/workflows/Pull%20Target%20Files%20from%20Applanga%20on%20specific%20branch/badge.svg"
        },
        {
            "id": 12962576,
            "node_id": "W_kwDOGBq-Vc4AxcsQ",
            "name": "Pull Target Files from Applanga",
            "path": ".github/workflows/applanga-pull.yml",
            "state": "active",
            "created_at": "2021-09-08T18:11:09.000+02:00",
            "updated_at": "2021-09-08T19:11:18.000+02:00",
            "url": "https://api.github.com/repos/oaks-view/github-workflow-example/actions/workflows/12962576",
            "html_url": "https://github.com/oaks-view/github-workflow-example/blob/master/.github/workflows/applanga-pull.yml",
            "badge_url": "https://github.com/oaks-view/github-workflow-example/workflows/Pull%20Target%20Files%20from%20Applanga/badge.svg"
        },
        {
            "id": 12962577,
            "node_id": "W_kwDOGBq-Vc4AxcsR",
            "name": "Push Source Files to Applanga",
            "path": ".github/workflows/applanga-push.yml",
            "state": "active",
            "created_at": "2021-09-08T18:11:09.000+02:00",
            "updated_at": "2021-09-08T18:11:09.000+02:00",
            "url": "https://api.github.com/repos/oaks-view/github-workflow-example/actions/workflows/12962577",
            "html_url": "https://github.com/oaks-view/github-workflow-example/blob/master/.github/workflows/applanga-push.yml",
            "badge_url": "https://github.com/oaks-view/github-workflow-example/workflows/Push%20Source%20Files%20to%20Applanga/badge.svg"
        }
    ]
}
```

From the response you can find the workflows in the `workflow` field and the `id` field of the entries is the `workflow_id`. Once the workflow id is gotten then the url can be put together like so
`https://api.github.com/repos/oaks-view/github-workflow-example/actions/workflows/24995310/dispatches` where `24995310` is the `workflow_id`.
In case the workflow name is used instead a sample url would look like so
`https://api.github.com/repos/oaks-view/github-workflow-example/actions/workflows/applanga-pull-branch.yml/dispatches`, where `applanga-pull-branch.yml` is the workflow file name. Please refer to the following screenshot 

![](https://www.applanga.com/assets/images/docu/webhook_branch_trigger_endpoint_url.png)

* **Headers** You need to set the following headers, `Authorization` and `Accept`. The `Authorization` header value is a valid github access token with repository permissions combined like so `token <Access token>`. For example if you had an access token `ghp_zWAdtUqmtFTY7qkqJS1wmuEx6t4X0SpIPv` then the Authorization header would be set like so `token ghp_zWAdtUqmtFTY7qkqJS1wmuEx6t4X0SpIPv`. Please refer to the following github [documentation](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) on how to generate an access token.
![](https://www.applanga.com/assets/images/docu/webhook_branch_trigger_headers.png)

* **Body** Click the **Body** tab and select **JSON**. A raw JSON text will be pasted in the textbox that contains the field `ref` which should be set to the name of the branch in which the workflow is intended to be triggered. For example if the workfow should be triggered in a branch named `staging` then the text to be pasted would be as follows

```
{
  "ref": "staging"
}
```

Next click **Save endpoint**.

![](https://www.applanga.com/assets/images/docu/webhook_branch_trigger_body.png)

For more information about triggering a workflow dispatch event via REST Api check [here](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event).

---
# Applanga Configuration

Because the workflows make use of the [Applanga Command Interface](https://github.com/applanga/applanga-cli) you also need to add a [.applanga.json](https://github.com/applanga/github-workflow-example/blob/master/.applanga.json) configuration file to your repository. 

To uniqly identify your project you need to provide your ***API Access Token*** which can be found on the **API** section in the Applanga Project Settings in the [Applanga Dashboard](https://dashboard.applanga.com). Select your project, click ***Project Settings***, click on **“Show API Token”** and copy the Token. The token can be stored as a [Encrypted Repository Secret](https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets#creating-encrypted-secrets-for-a-repository) labeled **APPLANGA\_ACCESS\_TOKEN** or as plain text directly in the config e.g.: `"app": { "access_token": "<YOUR_API_TOKEN>", ... }`.

The included config is set-up to push all changes to `translations.json` file under the directory `react_json_sample/en/` to Applanga and pull all other languages from Applanga as well as create the needed directorys on the daily schedule or via the commandline request as described above.

The example configuration is set-up to use the `"react_simple_json"` file format but Applanga supports a wide variety of other file formats and folder structures. Also the example works with only one file per language if you need to support more you need to provide a `tag` per file for details on that and more see the [Applanga CLI Documentation](https://github.com/applanga/applanga-cli).

```json 
{
  "app": { 
    "pull": {
      "target": [
        {
          "file_format": "react_simple_json", 
          "exclude_languages": ["en"],
          "path": "./react_json_sample/<language>/translations.json"
        }
      ]
    }, 
    "push": {
      "source": [
        {
          "language": "en",
          "file_format": "react_simple_json", 
          "path": "./react_json_sample/en/translations.json"
        }
      ]
    }
  }
}
```                                                                  
                                                               
