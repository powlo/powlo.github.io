Title: How to publish a pelican blog to GitHub
Date: 2025-04-25 17:31
Tags: pelican, python, github

**Difficulty:** Easy

**Time required:** 30 minutes

**Target audience:** Software developers

In the post I'll take you through the steps needed to publish a personal Pelican site from a GitHub repository. We'll do this using GitHub workflows so that a code push automatically updates the site. This post assumes some basic familiarity with software development tools and processes, particularly around git and github. Code examples here will assume a linux-like environment.

[Pelican](https://getpelican.com/) is a static site generator powered by Python with a wealth of community developed plugins and themes making it ideal for static content such as documentation or blogs.

Let's get started.

## 1. Prepare GitHub to host your content.

First we have to create a repository to store our site content. Go to your github profile and click the repositories tab. Then click "new":

![Create New Repository]({attach}images/2025-04-25/new-repository.png)

Since we're creating a site associated with your github user profile, GitHub rules dictate that the associated repository must be named `<yourusername>.github.io`.

![Name Repository]({attach}images/2025-04-25/name-repository.png){: style="max-width:25rem"}

There's no need to add a README or ignore file since we'll push the first commit later.

Finally "Create Repository" at the bottom of the page.

To tell GitHub that we want this repository to be deployed to the GitHub Pages we have to make some changes in the repository's settings:

![Settings]({attach}images/2025-04-25/settings-button.png){: style="max-width:30rem"}

Click "pages" in the sidebar.

![Settings Options]({attach}images/2025-04-25/settings-options.png){: style="max-width:20rem"}

And under "Build Deployment" select "GitHub actions".

![Build Source]({attach}images/2025-04-25/build-source.png){: style="max-width:15rem"}

We'll add this workflow in the final stage.

You can read more about GitHub Pages [here](https://docs.github.com/en/pages/quickstart).

## 2. Create the initial pelican site.

Using terminal create a development workspace

    :::bash
    $ mkdir yourusername.github.io
    $ cd yourusername.github.io

Create the initial git repository and set the upstream origin to what we created in the previous step.

    :::bash
    git init
    git remote add origin git@github.com:yourusername/yourusername.github.io.git

And create a virtual environent:

    :::bash
    $ python3 -m venv .venv
    $ source .venv/bin/activate
    $ pip install pelican[markdown] typogrify

Now initialise your site:

    $ pelican-quickstart

Enter values that are particular to your site (your name, the site's name etc) and accept all defaults, other than questions about URL prefix and deployment to GitHub Pages:

- When prompted, enter the URL prefix that will be applied to your site:

        :::bash
        > What is your URL prefix? (see above example; no trailing slash) https://yourusername.github.io

- Select `y` to deploy to GitHub Pages:

        :::bash
        > Do you want to upload your website using GitHub Pages? (y/N) y
        > Is this your personal page (username.github.io)? (y/N) y

Once completed you will have an initial project as follows:

    $ tree .
    .
    ├── content
    ├── Makefile
    ├── output
    ├── pelicanconf.py
    ├── publishconf.py
    └── tasks.py

`pelicanconf.py` contains site configuration while `publishconf.py` contains configuration specifically for the production environment (ie the deployed blog). The values you entered in the quickstart can be edited in these conf files if required.

`content` will store all your articles, `output` will contain rendered html while `tasks.py` and `Makefile` will contain functions that are used to generate site content (ie complete rendered html).

You can now run the local development server

    :::bash
    $ pelican --listen --autoreload --relative-urls

And visit http://127.0.0.1:8000 to see the first draft of the site.

Pelican supports content written in markdown (.md), reStructuredText (.rst) and HTML out of the box. Here are some useful links on [Markdown](https://www.markdownguide.org/) and [reStructuredText](https://www.sphinx-doc.org/en/master/usage/restructuredtext/basics.html).

## 3. Choose a theme.

To add a theme to your Pelican site you need to clone the `pelican-themes` repository:

    :::bash
    $ git clone --recursive https://github.com/getpelican/pelican-themes ~/pelican-themes

Then use `pelican-themes` to make your chosen theme available to be used:

    :::bash
    $ pelican-themes --install ../pelican-themes/Flex

Finally, reference your theme in your site's `pelicanconf.py`:

    :::python
    # pelicanconf.py
    THEME = "Flex"

(Note how theme names are case sensitive.)

Hit refresh in the browser and confirm that your theme has been applied.

# 4. Automate!

Let's add code to automate the site deployment.

Create a github workflow file named `.github/workflows/pelican.yml`

And add the following contents.

    #!yaml
    # .github/workflows/pelican.yml
    name: Deploy to GitHub Pages
    on:
    push:
        branches: ["master"]
    workflow_dispatch:
        inputs:
            deploy:
                required: false
                default: true
                description: "Check to deploy the site after a successful build."
                type: boolean
    jobs:
        deploy:
            uses: "getpelican/pelican/.github/workflows/github_pages.yml@main"
            permissions:
                contents: "read"
                pages: "write"
                id-token: "write"
            with:
                settings: "publishconf.py"
                requirements: "pelican[markdown] typogrify"
                theme: "https://github.com/alexandrevicenzi/Flex.git"
                deploy: ${{ (github.event_name == 'workflow_dispatch' && inputs.deploy == true) || (github.event_name == 'push' && github.ref_type == 'branch' && github.ref_name == github.event.repository.default_branch) }}

This uses a deploy template provided by Pelican to automate deployment. If you used a theme, you'll need to add the github url (https) as shown on line 23 above.

Tip: `workflow_dispatch` allows you to run the action manually (through the github.com UI) from any branch. You can verify that the build works _without_ deploying by setting workflow's input `deploy` false.

At this point it might be useful to add a .gitignore file to instruct git to ignore certain files, or directories. We definitely don't want to commit any content in the `/output/` directory that may have been generated during development:

    :::bash
    # .gitignore
    output/
    __pycache__/

Add all the files that we want to be on the site.

    :::bash
    $ git add .
    $ git commit 

Push the code

    $ git push origin master

And verify that the workflow completed successfully.

![Workflow Success]({attach}images/2025-04-25/workflow.png){: style="max-width:30rem"}

And finally, visit your deployed site.

Congratulations!