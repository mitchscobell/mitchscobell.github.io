# README.md

## These are basically notes to myself :)

[Forked from jekyll-now](https://github.com/barryclark/jekyll-now)

### Initial setup

Running locally I use the WSL and run to install the tools:

``` sh
sudo apt-get update -y && sudo apt-get upgrade -y

sudo apt-add-repository ppa:brightbox/ruby-ng

sudo apt-get update

sudo apt-get install ruby2.5 ruby2.5-dev build-essential dh-autoreconf

```

Next I run the website:

``` sh
sudo jekyll serve --trace
```

Then open http://127.0.0.1:4000/

(sudo needs to be used because my permissions are all jacked up in wsl for ruby and jekyll and I'm not going to bother fixing them since this is the only project I use them for)