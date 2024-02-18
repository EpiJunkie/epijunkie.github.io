---
title: Display formatted JSON via command line.
tags: [json, cli, python3, jq]
style: border
color: primary
description: Various ways to manipulate JSON with CLI tools.
comments: true
---

Here are a few ways to display JSON in readable format via command line as an alternative to using [JSONlint.com](https://jsonlint.com/) or similar online tools.

## Using Python 2

	Pipe to:
	python -m json.tool

	Add to .bashprofile:
	alias jpp='python -m json.tool'

## Using Python 3

	Pipe to:
	python3 -c "import json; import sys; print(json.dumps(json.loads(sys.stdin.read()), indent=4))"

	Add to .bashprofile:
	alias jpp='python3 -c "import json; import sys; print(json.dumps(json.loads(sys.stdin.read()), indent=4))"'

## Using jq

"jq is like `sed` for JSON data - you can use it to slice and filter and map and transform structured data with the same ease that `sed`, `awk`, `grep` and friends let you play with text." ~ per [jq's homepage](https://stedolan.github.io/jq/).

	Install:
	user@osx$ brew install jq

	Pipe to:
	jq
