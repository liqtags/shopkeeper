# The Complete Guide to Shopkeeper

Like all good tales, we must begin with a call to adventure. The adventure of
Shopkeeper begins with [The Beyond Group](https://thebeyondgroup.com) needing
to maintain three expansions stores using a single theme and wanting to use a
build system to support the development of a theme for a client.

Unfortunately, the Shopify GitHub integration work in this case. It expects a
single usage of a theme to be represented by a single repo. It will
automatically open PRs as customizations are made and there is no way to
separate a theme's data from its files. 

We want to use a single theme and repo for all three expansion stores to
reduce development effort. We want a development workflow that parallels
something you might use when doing custom full-stack web development. We want
to open PRs for changes, have preview environments created and updated
automatically with each commit pushed, and for preview environments to be
cleaned up once PRs are merged.

The Shopify [CLI](https://shopify.dev/docs/themes/tools/cli/) provides a
wonderful set of primitive interactions with a single store and a single theme.
With this foundation, we can build a workflow and more sophisticated commands
to solve the problem.

We decided to use GitHub Actions for CI/CD and to build Shopkeeper to encapsulate
the common operations the workflow previously described would require.

The first step is to establish a way to mange theme settings. Theme settings
are important because they are the data that defines how a storefront looks.
Mess them up and the storefront is broken. How to sync settings from the live
theme to a new theme is something that theme developers have to consider as
they work. 

In our case, we need a way to store multiple groups of settings, one per
store.

## Manage Settings
To manage theme settings we create buckets of settings. These buckets of settings
are kept in a `.shopkeeper` directory in the root of your project.

Run `shopkeeper bucket init` to create this directory. To add a `production`
bucket, run `shopkeeper bucket create production`.

```console
.shopkeeper
├── production
│   ├── config
│   ├── sections
│   ├── templates
│   ├── .env
│   └── .env.sample
└── .current-bucket
```

Each each folder in the bucket references the folders where a `.json` file
might be found in a theme.

For example, we create a directory called `client-x-theme`, we can bootstrap a
theme project using [Dawn](https://github.com/shopify/dawn):

```console
mkdir client-x-theme
cd client-x-theme
npm init
npm install --save-dev @shopify/cli @shopify/theme @thebeyondgroup/shopkeeper
npx shopify theme init --path theme

```

We add a `package.json` file and add the CLI tools we'll use. We create the
theme files in subdirectory named `theme`. You could name it something else if
you prefer. Just be sure to use that name whenever a command requires a path.

Next, we initialize Shopkeeper and create a bucket to hold our production settings:

```console
npx shopify bucket init
npx shopify bucket create production
```
We also install [direnv](https://direnv.net) in our shells so our shell environment variables are
updated anytime we update our `.env` file.

Run the following to configure `direnv`:
```console
echo "dotenv .env" > .envrc
direnv allow
```

Why we do this will be come clear in the next steps. Our final bit of setup is
to edit `.shopkeeper/production/.env` to contain:

```console
SHOPIFY_CLI_THEME_TOKEN="<your theme access password>"
SHOPIFY_FLAG_STORE="<your store url>"
SHOPIFY_FLAG_PATH="theme"
```

By setting these variables, we can omit them when calling CLI commands. Shopify CLI flags have
a corresponding environment variable. If you set it, it provides the value to
the flag and you don't need to pass the flag.

Assuming we have the correct store and password, we can now run:

```console
shopify bucket switch production --nodelete
```

This copies the `.env` from `.shopkeeper/production` into our project
directory, which `direnv` reloads. We pass the `--nodelete` command so we don't clobber
the settings currently in the repo. Now, we can run the following to pull down
the latest settings from our live theme:

```console
shopify theme settings download
```
This pulls down the latest settings into `theme`. We can then store these in
our `production` bucket by running:

```console
shopify bucket save production
```

We recommend adding the following lines to a project's `.gitignore`:

```gitconfig
# Shopkeeper #
##########
.shopkeeper/**/**/.env
.shopkeeper/.current-bucket
.env

# Theme #
##########
theme/assets
theme/config/settings_data.json
theme/templates/**/*.json
theme/sections/*.json
```

Adding this config makes only the settings in `.shopkeeper` trackable by `git`.
Doing so makes the bucket the source of truth for settings &mdash; an key part
of automating deployments.

Let's look at a couple more commands before combining these tools with GitHub Actions.

To switch between buckets, run:

```console
shopify bucket switch <bucket name>
```

Sometimes you'll want to update the settings in `theme` to those in a bucket.
To do that, run:

```console
shopify bucket restore <bucket name>
```

Each of `bucket save`, `bucket restore`, and `bucket switch` can be passed a
`--nodelete` flag. When you pass this flag settings files at the destination
are not removed first. Files from the source are added and pre-existing files
are overwritten with the file at the source.

With this knowledge, let's see how to use Shopkeeper to build CI/CD for themes.

## Deployment

Shopkeeper makes it straightforward to set up CI/CD using GitHub Actions. The
same pattern can be adapted to other source code management systems.

Let's begin with running deployments locally and then we'll apply these
concepts to GitHub Actions.

As a Shopify theme developer, you are likely familiar with using `shopify push`
to upload code to a theme. It's a low-level command that doesn't take settings
into account.

Shopkeeper supports multiple deployment strategies:

* Basic
* Blue/Green

By default, Shopkeeper assumes you want use a [blue/green deployment
strategy](https://en.wikipedia.org/wiki/Blue–green_deployment). A blue/green
deployment strategy in the context of Shopify theme development
means alternating between a blue and a green theme. One theme is live and the
other we refer to as on-deck. For example, using this approach, if a the blue
:large_blue_circle: theme is live, the green :green_circle: theme is on-deck.
When updates are to be made to a storefront, the on-deck receives the changes
and when it is ready, it is published.

The advantage of a using a blue/green strategy is that you won't end up with a
proliferation of themes in the admin. You will need just two themes. While
Shopify Plus stores can have many unpublished themes, without some organization
it's easy to lose track of what changes are contained in each theme.

To setup your project for blue/green deployments, we need to create a second
theme. We'll designate to the live theme as blue and this new theme as green.

```console
shopify theme push --unpublished --theme green
```

We'll list the themes on our store using:

```console
shopify theme list
```

Take note of the IDs for the live :large_blue_circle: theme and our newly
created `green` :green_circle: theme. Next, we'll add these two IDs to our
`.env` in `.shopkeeper/production`:

```console
SHOPIFY_CLI_THEME_TOKEN="<your theme access password>"
SHOPIFY_FLAG_STORE="<your store url>"
SHOPIFY_FLAG_PATH="theme"
SKR_FLAG_GREEN_THEME_ID=<green id>
SKR_FLAG_BLUE_THEME_ID=<blue id>
```

By setting these envronment variables we eliminate our need to pass the `--green` and `--blue` flags
each time we run `shopify theme deploy`. Also, by putting these values in our `.env`, when we switch buckets
our whole development context is updated.

When you run `shopkeeper theme deploy`, Shopkeeper will:

1. Download settings from the live theme
2. Push code to the on-deck theme
3. Rename the on-deck theme to be `[<HEAD_SHA>] Production - <color>`

Don't worry about updating the name of the live theme. It will be updated when it becomes the on-deck
theme. Also, note this command does not automatically publish the on-deck
theme. Quite often you'll want to do take a moment to run final checks before
publishing the theme.

When you're ready, you can publish the on-deck theme by running:
```console
shopify theme publish --theme $SKR_FLAG_<color>_THEME_ID
```
You can also publish it from the admin.