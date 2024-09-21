---
title: Blogging with Hugo & Cloudflare Pages
date: 2024-09-21
description: |
    After I recently moved my blog to Hugo and published it via Cloudflare Pages, I'll show you how in this post.  
categories:
    - services
tags:
    - blog
    - hugo
    - cloudflare
    - git
    - github
    - domain
---

## Intro

When you think of blogs on the Internet, Wordpress is the first thing that comes to mind. I also hosted and used it myself for a while because it was convenient, accessible from anywhere and easy to expand thanks to plugins. However, Wordpress is relatively heavy, I don't need many of its functions and as it is one of the most-known CMSs, the potential for attacks is correspondingly widespread. That's why I decided to switch to [Hugo](https://gohugo.io/). 

Hugo is an open source framework for generating static pages. The content is written in Markdown and Hugo uses it to create a static HTML website that can be hosted on any web server. As the pages are pre-generated, access is faster. Thanks to Git, everything is versioned, and the website can be easily deployed via [Github Pages](https://pages.github.com/) or [Cloudflare Pages](https://pages.cloudflare.com/) if you use Github and don't want to host the content yourself. 

I opted for Cloudflare Pages on the free plan. Some of my reasons were:
- I already use Cloudflare for the DNS management of my domain
- The content is served via the Cloudflare CDN
- Unlimited static requests and bandwidth
    - Github Pages have a limit of 100 GB / month in the free plan. Even if this has to be reached first, it is a limit that should be kept in mind
- Integration with Github and Gitlab. With every `git push` on the configured branch, the build process is triggered and the finished page is published

Even if you cant't create and edit content via the web interface like with Wordpress, Hugo is not particularly complex. However, you should have a basic understanding of git and not be afraid of the console.

## Create the blog locally

Git and Hugo should be installed on the system. On Linux, Git should usually be pre-installed and Hugo should be available via the package manager, on Windows it is recommended to use the **Windows Subsystem for Linux (WSL)**. On Windows I use e.g. Ubuntu 24.04 LTS via WSL, with my Linux systems I use Hugo directly.

> **Note:** 
> 
> With WSL, the Windows drives are available under `/mnt`. If you open WSL from Explorer or via cmd, you are still in the corresponding folder, e.g. `/mnt/c/Users/<username>`. If you create the folder for the blog in a folder mounted by Windows, Hugo's live preview will not work as file changes are not tracked. Instead, a folder should be created directly in the WSL file system, for example under `~/blog`.

After navigating to the parent folder, execute the following command. For example, `blog` is the name of the folder that should be created and can be chosen as desired:

```
hugo new site blog
```

With this command, hugo creates the corresponding folder structure and various files. After switching to the created folder, initialize the empty Git repository there:

```
git init
```

The only thing missing now is a theme. Hugo offers an [overview of different themes with preview images](https://themes.gohugo.io/), the tags can be used to filter the desired functionality. I opted for [Stack](https://themes.gohugo.io/themes/hugo-theme-stack/) for the time being, but if I don't like it, the theme can be easily replaced without having to change the content of the website.

The source code behind the themes offered on the overview page can be found in repositories on Github. These can be added as [Git Submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules). In my case with Stack:

```
git submodule add https://github.com/CaiJimmy/hugo-theme-stack.git themes/stack
```

Finally, you only need to specify which theme is to be used in the configuration. To do this, the following line must be added to the `hugo.toml` file: 

```
theme = 'stack'
```

> **Note:** 
> 
> This configuration file is kept relatively simple. It can be used to further configure Hugo and the theme. Instead of a single file, you can also use a folder structure. For more information, see [the Hugo documentation](https://gohugo.io/getting-started/configuration/).
>
> A [template for the configuration files](https://github.com/CaiJimmy/hugo-theme-stack-starter/tree/master/config/_default) can be found in the Quick Start Repo of Stack.

You can then start the Hugo development server and display the website, even if there is no content yet:

```
hugo server
```

## The Github repository

Next, a new repository is created on Github. The name can be chosen as desired, but the repo does not need to be initialized.

One advantage of Cloudflare Pages is that the repo can also be private - with Github Pages in the free plan, the visibility must be *public*. 

Locally index all files created for the first commit, create it, add the previously created repository as a remote and upload the commit. The `<destination>` is either `https://github.com/UserName/RepoName.git` (HTTPS) or `git@github.com:UserName/RepoName.git` (SSH):

```
git add *
git commit -m "Initial Setup"
git branch -M main
git remote add origin <destination>
git push -u origin main
```

## Cloudflare Pages

The next step is to generate the blog statically and make it available via Cloudflare Pages. To do this, click on “Workers & Pages > Overview” in the Cloudflare dashboard and then on the “Create” button. If you switch to the “Pages” tab there, you can import an existing Git repository from Github or Gitlab. You can also upload the files directly, but you would have to do this manually every time you make a change.

After logging in, select the account and the repository to be tracked. Then select a project name (this name is used to generate a permanent host name on pages.dev), the branch and configure the build setting. You can select **Hugo** as the preset, which also fills out the build command (`hugo`) and the output directory (`public`). I modify the build command as follows:

```
hugo --minify -b $BASE_URL
```

Under “Environment variables (advanced)” I now configure the environment variable `BASE_URL` with the value `https://www.akumatic.eu`. If I create a preview environment later, I can then use the value `$CF_PAGES_URL` for this. If you don't want to use your own domain, you can also use `$CF_PAGES_URL` directly here.

In addition, Cloudflare Pages uses an older version of Hugo (in my case v.0118.2), which throws an error when rendering with the current version of *Stack* (`<.Site.Lastmod.IsZero>: can't evaluate field Lastmod in type page.Site`). There is an [Issue on Github](https://github.com/CaiJimmy/hugo-theme-stack/issues/974) for this problem, as a solution you have to explicitly set the version of Hugo to a version >= 0.123.0. For this I have created the variable `HUGO_VERSION` with the value `0.123.8`.

If you continue now, Cloudflare builds the page and publishes it. The blog can be accessed directly with the displayed subdomain of `.pages.dev`.

### Use your own domain

However, you may already have rented your own domain and want to use it directly or a subdomain of it. In my case, I would like to use the subdomain `www` for the website itself and redirect from the domain apex to `www`. 

### Subdomain `www`

In the dashboard under “Workers & Pages”, select the page you have just created and switch to the *Custom domains* tab. After clicking on “Set up a custom domain”, you can enter the desired domain to be used for the website - in my case `www.akumatic.eu`. The DNS entry to be created is then displayed.

After confirmation and a short wait, the entry is set and the certificate is generated - the site can now be accessed via the domain.

### Redirect from the Apex

Similar to `www`, I could also configure the Apex domain so that the website would be directly accessible via both domains. However, to keep the website consistent, I only set up `www` and redirect requests from `akumatic.eu` to `www.akumatic.eu`. 

A simple CNAME entry is not sufficient here, as the SSL certificate was only issued for `www.`. A **redirect rule** is therefore used, for which the DNS entry for the Apex domain must be set up. In my case, the entry looks like this: 

| Property | Content | 
| -- | -- |
| Type | CNAME |
| Name | `akumatic.eu` |
| Content | `www.akumatic.eu` | 
| Proxy status | Proxied |

> **Note:** 
> 
> Normally you cannot use CNAME for the Apex entry, Cloudflare makes this possible by [CNAME Flattening](https://developers.cloudflare.com/dns/cname-flattening/)

The *Redirect Rule* can then be configured. In the Cloudflare dashboard for the domain, go to `Rules > Redirect Rules` and create a new one. The name can be chosen as desired. The following settings are made so that the full path is retained in the event of a redirect:

##### If...

- [ ] Wildcard pattern
- [ ] All incoming requests
- [x] Custom filter expression

| Property | Content | 
| -- | -- |
| Field | Hostname |
| Operator | equals |
| Value | `akumatic.eu` | 

##### Then...
URL redirect

| Property | Content | 
| -- | -- |
| Type | Dynamic | 
| Expression |  `concat("https://www.akumatic.eu", http.request.uri.path` |
| Status Code | 301 | 
| Preserve query string | :ballot_box_with_check: | 

## Comments with Giscus

*Stack* offers several options for comments, including Giscus. Giscus has the advantage that comments are realized via the discussion function of Github and no secret has to be stored in the configuration. 

To do this, the [discussion function must be activated for the repo](https://docs.github.com/de/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/enabling-or-disabling-github-discussions-for-a-repository) and [Giscus must be installed](https://github.com/apps/giscus). You can choose whether it should be installed for all or only selected repos. 

I have also tidied up the existing categories in the Github discussions and created a new category explicitly for comments of the “Announcement” type - this means that only I and the Giscus app can open new discussions there.

On the [Giscus page](https://giscus.app/) under “Configuration”, you can check whether all requirements have been met. You can then select the options on the page as you wish and find a ready-made code block for integrating Giscus under “activate giscus”. However, we are only interested in the values that we enter in the configuration of Hugo. The corresponding section can be found in the template under [config/_default/params.toml](https://github.com/CaiJimmy/hugo-theme-stack-starter/blob/master/config/_default/params.toml):

```
## Comments
[comments]
enabled = true
provider = "giscus"

[comments.giscus]
repo = ""
repoID = ""
category = ""
categoryID = ""
mapping = ""
lightTheme = ""
darkTheme = ""
reactionsEnabled = 1
emitMetadata = 0
```

After filling in the values and creating a post, the functionality can be tested directly.
