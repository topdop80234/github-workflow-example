# Applanga Github Workflow Integration Example
 
The example [repository](https://github.com/applanga/github-workflow-example) showcases a full cycle [github workflow setup](https://github.com/applanga/setup-applanga-cli) integration with the [Applanga Localization Platform](https://www.applanga.com).

The benefit of using github workflows is that you can automate your localization process without the need to share any repository credentials with your localization provider.

To use github workflows on your repository you need to create a folder call .github/workflows/ and place the workflow configuration .yml files in there. 

For a more detailed introduction to github worklfows please see the [Github Documentation](https://help.github.com/en/actions/configuring-and-managing-workflows/configuring-a-workflow)

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
      - uses: applanga/setup-applanga-cli@v1.0.0
        with:
          version: 1.0.47
      - name: Push Sources to Applanga
        run: applanga push --force
        working-directory: checkout
```

[.github/workflows/applanga-pull.yml](https://github.com/applanga/github-workflow-example/blob/master/.github/workflows/applanga-pull.yml) checks once per day if there are new translations available from Applanga and then creates a pull request in the repo with the newly added languages or updated translation files. 

```yaml
name: "Pull Target Files from Applanga"
on:
  schedule:
   - cron:  '* 23 * * *'
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
      - uses: applanga/setup-applanga-cli@v1.0.0
        with:
          version: 1.0.47
      - name: Pull translations from Applanga
        run: applanga pull
        working-directory: checkout
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v2
        with:
          branch: newTranslations
          commit-message: Updated translations
          title: Updated translations 
          body: Pulled in new translations from Applanga portal
          path: 'checkout'
```

---
## Manual Translation Pull
**Optionally** the `applanga-pull` workflow can also be triggered manually through a [Github REST API](https://developer.github.com/v3/repos/#create-a-repository-dispatch-event)`repository_dispatch` *POST* request with `"event_type":"applanga-pull"`. This could be done from the commandline like this:

* Ensure `curl` is installed. (https://curl.haxx.se/)
* In the terminal of your choice execute this `curl` request, replacing `:owner` and `:repo` with the appropriate values for your repository and `$PERSONAL_ACCESS_TOKEN` with a [personal access token](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line). You must use a personal access token with the `repo` scope.

```shell
curl -v -H "Accept: application/vnd.github.everest-preview+json" \
        -H "Authorization: token $PERSONAL_ACCESS_TOKEN" \
        https://api.github.com/repos/:owner/:repo/dispatches \
        -d '{"event_type":"applanga-pull"}'
```
***NOTE:*** *Applanga makes its files available through a cdn so after changes on the dashboard you need to wait for 10+ minutes until the updates are available in the target files.*

---
# Applanga Configuration

Because the workflows make use of the [Applanga Command Interface](https://github.com/applanga/applanga-cli) you also need to add a [.applanga.json](https://github.com/applanga/github-workflow-example/blob/master/.applanga.json) configuration file to your repository. 

The configuration needs a `"access_token"` which uniqly identifies your project and can be found on the **API** section in the Applanga Project Settings in the [Applanga Dashboard](https://dashboard.applanga.com). Select your project, click ***Project Settings***, click on **“Show API Token”** and copy the Token.

The included config is set-up to push all changes to `translations.json` file under the directory `react_json_sample/en/` to Applanga and pull all other languages from Applanga as well as create the needed directorys on the daily schedule or via the commandline request as described above.

The example configuration is set-up to use the `"react_simple_json"` file format but Applanga supports a wide variety of other file formats and folder structures. Also the example works with only one file per language if you need to support more you need to provide a `tag` per file for details on that and more see the [Applanga CLI Documentation](https://github.com/applanga/applanga-cli).

```json 
{
  "app": {
    "access_token": "5ece6f6a9f62210da008fb62!d43f18f2f4b183e45f1cc6a4836ca8f0", 
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
                                                               
