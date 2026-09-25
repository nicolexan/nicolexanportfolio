---
title: Earnest Gumiho - Product Landing Page with AI Pair Programming
published: 2026-09-25
description: "Describing how I utilized Claude Code in completing a client's landing page."
image: "earnestgumiho.png"
tags: ["Project", "AWS", "AI"]
category: Client Projects
draft: false
---

## Overview
My client had a need to host their website on their domain that was purchased with a particular DNS registrar, except that this registrar charges an exoborant amount of money to keep their company page up. The goal was simple, to build a product showcase page that would cut their annual cost by 55%. 

## Build Environment and Setup
I natively use a Windows machine so I utilized VS Code for editing the source code and synced it over to Ubuntu on WSL to conduct local builds and testing. This ensures I can still use the development environment I'm comfortable with. I also used an Astro 7 template (as you can tell, I'm a big fan of Astro).

```bash
rsync -a --delete   --exclude node_modules --exclude .astro --exclude dist   --exclude pnpm-workspace.yaml   ~/OneDrive/Documents/gumihollc/ ~/gumihollc/
```
If I had updated any packages or libraries then I would install them again and run the build server on `http://localhost:4321/`
```
pnpm install
pnpm dev
```
From here I could show my client what it was looking like so far and collect a list of change requests.

![A closer look at the product page](./productpage.png)

## AI Pair Programming
I owned the requirements in plain language and the decisions made when editing the template to fit my client's needs; the AI assistant searched the code, made edits, and ran the pipeline. I sent related edits together, and they were built and smoke-tested as a single unit before showing the client. I also provided the AI constraints and paused when there needed to be decisions made (*AKA not letting the AI run wild)*. Every change was checked with a real build and route tests, and failures were reported with their output.

The AI was particularly useful when learning an Astro template. Astro builds in islands making it a learning-curve to really understand how a website is built. The AI streamlined this learning-curve and I felt a lot more comfortable making minor edits throughout the project.

Regarding content, AI still needs a lot of input which I provided from their original webiste in the form of a `.md` file during the beginning stages of the project. Once most of the information was propagated throughout the webpages, then I could remove that extraneous file.

Claude also caught a mistake in one of the images I provided, where I had previously downloaded an extra mountain range near a shoreline image and confused it with my client-provided image of a mountain range in Geumsan. The AI mentioned that Geumsan is deep within the Korean peninsula with no shorelines to be seen anywhere and that the image was from elsewhere. It was a good catch, and I was really surprised that it caught this!

## CI & GitHub Actions
The CI pipeline is essentially the same as what I would run in my local environment. This workflow is ran whenever changes are pushed to main, which verifies the work I was doing. At some point, I would like to add a functionality to stop the deployment of the website if this CI check fails. This is also helpful as a check for PRs since I am also utilizing dependabot to automatically create PRs when packages are updated.

```yaml
name: CI

on:
  push:
    branches: [main, master]
  pull_request:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Format check
        run: pnpm format:check

      - name: Typecheck and build
        run: pnpm build

      - name: Smoke tests
        run: pnpm test:smoke
```

## AWS Amplify
To keep deployments as simple as possible I opted to use AWS Amplify as I've used it previously and found it the easiest method for this sort of job. To set it up, I just had to allow access to the specific gumihollc repository and select which branch to deploy from. Because of the connection to GitHub, whenever there's an update to the main branch, it'll automatically deploy the site. The total deploy time takes about ~3-4 minutes each time. 

## DNS Routing
Because the original DNS registrar also hosts their email setup linked to their domain, we decided to keep the domain registered with them to keep their easy access to the email configurations. To enable this, we had to configure their domain to use `custom nameservers`. On the AWS Amplify side, there is a option to use your own custom domain and to use a 3rd party DNS registrar. AWS provides four nameservers which can then be added to the original DNS registrar so that traffic to that domain can be routed to the AWS Amplify hosted website.


:::note[Reflection]
Completing this project definitely improved my confidence with AI assisted programming and it was really fun to watch it all come together.
:::

::github{repo="nicolexan/gumihollc"}
