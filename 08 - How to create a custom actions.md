## Creating a Custom Action

1. Create a New Private Repository:<br>
Start by creating a new private repository in your GitHub account where you'll store your custom action code.

2. Set Up Your Development Environment:<br>
Clone the private repository to your local environment using the following 
command:
    ```bash
    git clone <repository_url>
    ```
3. Create Your Custom Action Code (optional):<br>
Inside the cloned repository, create a new directory for your custom action. This directory will contain the code for your action.

    ```bash
    mkdir -p .github/actions/private-action
    ```

4. Write Your Action Code(optional):<br>
Write the code for your custom action. The action code typically includes scripts or commands(bash script or json file etc.) that you want your action to perform when executed. Ensure that the code is structured according to GitHub's requirements for actions.

    ```bash
        #!/bin/bash
        echo "Hello, world! This is a private action."
    ```
   _Create a shell script named run.sh inside the .github/actions/private-action directory._ 

5. Create an Action Metadata File:<br>
In the root of your action directory, create a file named  `action.yml (or action.yaml)` which contains metadata about your action. 

    E.g. _Create a file named action.yml inside the .github/actions/private-action directory. This file contains metadata about your action_

    ```yaml
        name: 'Private Action'
        description: 'A private GitHub Action'
        runs:
        using: 'composite'
        steps:
            - run: ./run.sh

    ```

6. Commit Your Changes:<br>
Once you've finished writing your action code and created the action.yml file, commit your changes to the repository:    
    ```sql
    git add .
    git commit -m "Add private action code"
    git push origin <branch>
    ```
7. Use Your Custom Action in Workflows: <br>
You can now use your private action in your GitHub workflows. In your workflow YAML file, reference your private action like this:

        ```yaml
        jobs:
        build:
            runs-on: ubuntu-latest
            steps:
            - name: Use Private Action
                uses: ./.github/actions/private-action
        ```
### Addition

#### Using tags for release management
1. Create Tags for Releases:<br>
Whenever you have a new release of your custom action, create a Git tag to mark the release version. For example, you can use semantic versioning (e.g., v1.0.0, v1.1.0, etc.) to tag your releases.

    ```shell
    git tag -a -m "Description of this release" v1
    git push --follow-tags
    ```
2. Update Workflow YAML Files:<br>
In your workflow YAML files, reference the specific version of your custom action using the tag. This ensures that your workflows always use a specific version of the action.
    ```yaml
    jobs:
    build:
    runs-on: ubuntu-latest
    steps:
      - name: Use Private Action
        uses: ./.github/actions/private-action@v1.0.0
    ```