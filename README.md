# Applanga Github Workflow Integration Example
 
The example [repository](https://github.com/applanga/github-workflow-example) showcases a full cycle [github workflow setup](https://github.com/applanga/setup-applanga-cli) integration with the [Applanga Localization Platform](https://www.applanga.com).

The benefit of using github workflows is that you can automate your localization process without the need to share any repository credentials with your localization provider.

To use github workflows on your repository you need to create a folder call .github/workflows/ and place the workflow configuration .yml files in there. 

For a more detailed introduction to github workflows please see the [Github Documentation](https://help.github.com/en/actions/configuring-and-managing-workflows/configuring-a-workflow)

---
# Workflow Configurations

The [repository](https://github.com/applanga/github-workflow-example) contains 2 workflows. Running and past workflow actions can be tracked under the [**Actions** Tab](https://github.com/applanga/github-workflow-example/actions).

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

[.github/workflows/applanga-pull.yml](https://github.com/applanga/github-workflow-example/blob/master/.github/workflows/applanga-pull.yml) pulls new translations available from Applanga and then creates a pull request in the repo with the newly added languages or updated translation files. For this to work a `webhook endpoint` has to be configured for the project on Applanga dashboard to trigger the workflow. See `Configure Webhook Endpoint` below for how to configure 
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

---
## Configure Webhook Endpoint
To the trigger the `applanga-pull` workflow a webhook endpoint has to configured in the projects settings page on applanga. 

The webhook is triggered at least 15 minutes from when there is no translation change. This means whenever translation is added or edited a webhook request is scheduled to be sent to all configured endpoints for the project 15 minutes later. The scheduled request will be sent as planned unless there is a new change to translation before the scheduled time, in which case it is rescheduled to be sent 15 minutes later.

The endpoint should be configured to match the following `Github REST API` post request:

```shell
curl -v -H "Accept: application/vnd.github.everest-preview+json" \
        -H "Authorization: token $PERSONAL_ACCESS_TOKEN" \
        https://api.github.com/repos/:owner/:repo/dispatches \
        -d '{"event_type":"applanga-pull"}'
```

Here are the steps to setup the `Webhook Endpoint`
* Login to Applanga dashboard and navigate to the project. Then click on `Project Settings`

![]({{site.baseurl}}assets/images/docu/groups_editapp.png)

* In the settings page scroll down to the section `WEB HOOKS` and click the `Add endpoint` button, this will show a modal
where the endpoint values can be entered
![]({{site.baseurl}}assets/images/docu/webhook_settings.png)

The values should be configured to match the preceding curl request. To be sure, please ensure the following
- `POST` is selected as http method
- `JSON` is selected in the body tab
- `{"event_type":"applanga-pull"}` is entered in the `Request body`

![]({{site.baseurl}}assets/images/docu/webhook_endpoint_header.png)

![]({{site.baseurl}}assets/images/docu/webhook_endpoint_body.png)

You can test the configured endpoint to make sure everything works well by clicking the `Test endpoint` button top right.
If the config is done correctly the test result should look like the screeshot below

![]({{site.baseurl}}assets/images/docu/webhook_endpoint_test.png)

Click `Save endpoint` and you're done!.

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
                                                               
