---
title: 'Safely Saving Secrets'
date: Tue, 10 Nov 2020 03:41:43 +0000
draft: false
tags: ['bash', 'security', 'shell']
---

Because alliteration. Moving on.

If you're interacting with APIs of any kind regularly, you probably have the credentials saved somewhere. Maybe you're already using a solution to securely store these, in which case congratulations, you're better than most. I, for one, was not. I assuaged my guilt with the knowledge that my Mac's disk encryption meant that they were protected, but the whole thing still felt icky.

This was briefly discussed in Slack, and [this method](https://grantorchard.com/securing-environment-variables-with-1password/) of dealing with the problem came up. In short, it uses 1Pass to store secrets, and their CLI to access them and load them into the shell environment. That was all well and good, but I wanted a way to programmatically create the entry in the first place. 1Pass' templates are JSON, so this wasn't overly difficult with the help of jq. Download the CLI [here.](https://1password.com/downloads/command-line/)

First, you can get a blank template from the CLI, but it won't have the fields needed, so I'll paste one below. Add further blocks as needed - note the incrementing counter in n. This has to have some value, and it must be unique. I have no idea what its purpose is internally to 1Pass, but a number works fine. v will be filled in later. The section name was pulled from one that was manually created and then retrieved using the CLI - your guess is as good as mine.

```
# template_edit.json
{
  "notesPlain": "",
  "password": "",
  "passwordHistory": [],
  "sections": [
    {
      "fields": [
        {
          "k": "concealed",
          "n": "0",
          "t": "FIRST_SECRET",
          "v": ""
        },
        {
          "k": "concealed",
          "n": "1",
          "t": "SECOND_SECRET",
          "v": ""
        }
      ],
        "name": "Section_f3ryg4fyem3k7ldzwknl4o3rza",
        "title": "ENV VARS"
    }
  ]
}
```

```
# build_1pass.sh

#!/usr/bin/env bash

# jq -c flattens the human-readable JSON above for use by op
cat template_edit.json | jq -c > template.json

# You can really leave the --generate-password out, as all it's doing is assigning a password to the "ENV VARS" entry,
# but I opted to leave it in to demonstrate the functionality
op edit item $(echo $(op create item Password $(op encode < template.json) --generate-password) | jq -r '.uuid') title="ENV VARS"

# There may well be a way to do this from within the template itself, but this works, assuming you named your
# keys in the template the same as the environment variables you're extracting
for i in $(jq -r '.sections[].fields[].t' template.json); do
    op edit item "ENV VARS" $i="${!i}"
done
```

Now, check 1Pass for the new entry. You should see something like this.

![](/images/2020-11-10-safely-saving-secrets/0.png)

```
# .get_1pass.sh
# Most of this is taken straight from https://grantorchard.com/securing-environment-variables-with-1password/
# as linked at the beginning of this post. I added a login checker, an optional loader, and tweaked the JSON path.

#!/usr/bin/env bash

get_secrets() {
    if [[ ! $(op get item "ENV VARS" 2> /dev/null) ]]; then
        eval $(op signin $YOUR_SHORT_NAME)
    fi

    # My setup uses a 1Password type of 'Password' and stores all records within a
    # single section. The label is the key, and the value is the value.
    ev=$(op get item "ENV VARS")

    # Convert to base64 for multi-line secrets.
    # The schema for the 1Password type 'Password' uses t as the label, and v as the value.
    for row in $(echo ${ev} | jq -r -c '.details.sections[].fields[] | @base64'); do
        _envvars() {
            echo ${row} | base64 --decode | jq -r ${1}
        }
        echo "Setting environment variable $(_envvars '.t')"
        export $(echo "$(_envvars '.t')=$(_envvars '.v')")
    done
}

# Login to 1Password.
# Assumes you have installed the OP CLI and performed the initial configuration
# For more details see https://support.1password.com/command-line-getting-started/
echo "Load secrets? "

# As the file has to be sourced by your .{ba,z}shrc, the shebang isn't very useful. Since I use zsh,
# I can't use read -p since it has a different meaning than bash. This was a reasonable workaround for compatibility.
if [[ $SHELL =~ bash ]]; then
    read -n1 -s input
elif [[ $SHELL =~ zsh ]]; then
    read -k1 -s input
fi

if [[ $input == [Yy] ]]; then
    get_secrets
else
    echo "Skipping secrets"
fi
```

And there you have it! You'll have a minor annoyance of having to enter your master password with every new terminal (or skip it if you aren't going to be interacting with anything requiring those secrets), but you can sleep soundly knowing that if criminals [beat you with a](https://xkcd.com/538/) [wrench](https://xkcd.com/538/), they'll have to hit you at least twice.

