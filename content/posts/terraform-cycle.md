+++
title = "Terraform Cycle"
date = 2019-02-19T13:23:25Z
draft = false
description = "Fixing Terraform cycle errors using Terraform graph function"
slug = "terraform-cycle"
tags = ["terraform"]
categories = ["tech"]
externalLink = ""
+++

## Terraform Cycle error

Using Seth Vargo's project for running [`vault-on-gke`](https://github.com/sethvargo/vault-on-gke), I ran into the Terraform `Cycle` error when pulling in the latest changes from his repo (which we had forked). Both Terraform and I were happy with the `plan`, so I decided to go ahead with the `apply`. That's when I saw this:

![Terraform Cycle error](/images/terraform-cycle.png "Terraform Cycle error")

That's not particularly clear, so I decided to run `terraform graph -draw-cycles -type=apply`. This proved to be useless, because it didn't highlight any cycles at all. 

After some Googling, I ran across [this comment from the man himself, Mitchell Hashimoto](https://github.com/hashicorp/terraform/issues/11090#issuecomment-271173812). I gave it a go:

```bash
$ terraform plan -out tfplan
$ terraform graph -draw-cycles -type=apply tfplan \
  | dot -Tpng > graph.png
```

This proved much more useful (albeit still unclear)... However, I could now see where the cyclical dependencies were occuring in my Terraform!
